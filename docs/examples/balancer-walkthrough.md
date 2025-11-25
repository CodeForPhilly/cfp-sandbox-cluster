# Balancer Deployment Walkthrough

This document provides a complete, real-world example of deploying the Balancer application to the sandbox cluster.

## Table of Contents

- [Project Overview](#project-overview)
- [Repository Structure](#repository-structure)
- [Complete File Listings](#complete-file-listings)
- [Deployment Process](#deployment-process)
- [Architecture Decisions](#architecture-decisions)

## Project Overview

**Balancer** is a medication management application that helps balance medication interactions using AI.

- **Repository**: [CodeForPhilly/balancer-main](https://github.com/CodeForPhilly/balancer-main)
- **Technology**: Django (Python) backend serving React frontend
- **Deployment**: Single-container architecture
- **URL**: <https://balancer.sandbox.k8s.phl.io>

## Repository Structure

### Project Repository (balancer-main)

```
balancer-main/
├── deploy/
│   └── manifests/
│       └── balancer/
│           ├── base/
│           │   ├── kustomization.yaml
│           │   ├── namespace.yaml
│           │   ├── deployment.yaml
│           │   ├── service.yaml
│           │   ├── ingress.yaml
│           │   └── secret.template.yaml
│           └── overlays/
│               └── dev/
│                   └── kustomization.yaml
├── frontend/
│   ├── src/
│   ├── package.json
│   └── Dockerfile (dev)
├── server/
│   ├── balancer_backend/
│   ├── requirements.txt
│   └── Dockerfile.prod (legacy)
├── Dockerfile.prod  (ROOT - multi-stage build)
├── docker-compose.yml
├── docker-compose.prod.yml
└── .github/
    └── workflows/
        └── containers-publish.yml
```

### Cluster Repository (cfp-sandbox-cluster)

```
cfp-sandbox-cluster/
├── .holo/
│   ├── sources/
│   │   └── balancer.toml
│   ├── lenses/
│   │   └── balancer.toml
│   └── branches/
│       └── k8s-manifests/
│           └── balancer/
│               └── manifests.toml
├── balancer/
│   └── kustomization.yaml
└── balancer.secrets/
    └── balancer-config.yaml
```

## Complete File Listings

### Project Repository Files

#### `deploy/manifests/balancer/base/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

#### `deploy/manifests/balancer/base/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: balancer
```

#### `deploy/manifests/balancer/base/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: balancer
  name: balancer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: balancer
  strategy: {}
  template:
    metadata:
      labels:
        app: balancer
    spec:
      containers:
        - image: ghcr.io/codeforphilly/balancer-main/app
          name: app
          envFrom:
            - secretRef:
                name: balancer-config
          ports:
            - containerPort: 8000
          readinessProbe:
            httpGet:
              path: /admin/
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
```

**Key Design Choices:**

- Single container named `app`
- All configuration from `balancer-config` Secret
- Readiness probe on `/admin/` endpoint
- No resource limits (can be added later)

#### `deploy/manifests/balancer/base/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: balancer
  labels:
    app: balancer
spec:
  ports:
    - name: http
      port: 8000
      targetPort: 8000
  selector:
    app: balancer
```

**Key Design Choices:**

- ClusterIP service (default)
- Port 8000 for both service and target
- Simple label selector

#### `deploy/manifests/balancer/base/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: balancer
  annotations: {}
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - HOSTNAME_PLACEHOLDER
      secretName: balancer-tls
  rules:
    - host: HOSTNAME_PLACEHOLDER
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: balancer
                port:
                  number: 8000
```

**Key Design Choices:**

- Uses placeholder for hostname (patched by overlay)
- Empty annotations (added by overlay)
- Single catch-all path (Django handles all routing internally)
- TLS secret will be auto-created by cert-manager

#### `deploy/manifests/balancer/base/secret.template.yaml`

```yaml
# Secret Template for Balancer Application
#
# IMPORTANT: This file is a TEMPLATE only. Do NOT create a Secret manifest in this
# repository. Secrets should be created in each target cluster using cluster-specific
# tools (e.g., SealedSecrets in the cfp-sandbox-cluster).
#
# To create the secret manually in a cluster:
#   kubectl create secret generic balancer-config --from-literal=DEBUG="0" ...
#
# Required secret keys:
# -------------------
# DEBUG: "0" or "1" - Django debug mode (use "0" for production)
# SECRET_KEY: "your-secret-key-here" - Django secret key
# DJANGO_ALLOWED_HOSTS: "hostname1 hostname2" - Space-separated list
# SQL_ENGINE: "django.db.backends.postgresql" - Database engine
# SQL_DATABASE: "balancer_prod" - Database name
# SQL_USER: "balancer" - Database username
# SQL_PASSWORD: "your-db-password" - Database password
# SQL_HOST: "database-host" - Database hostname
# SQL_PORT: "5432" - Database port
# DATABASE: "postgres" - Database type
# LOGIN_REDIRECT_URL: "/" - URL to redirect after login
# OPENAI_API_KEY: "sk-..." - OpenAI API key
# PINECONE_API_KEY: "..." - Pinecone API key
# EMAIL_HOST_USER: "" - Email host username (optional)
# EMAIL_HOST_PASSWORD: "" - Email host password (optional)
```

#### `deploy/manifests/balancer/overlays/dev/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: balancer

resources:
  - ../../base

images:
  - name: ghcr.io/codeforphilly/balancer-main/app
    newTag: latest

patches:
  - target:
      kind: Ingress
      name: balancer
    patch: |-
      - op: add
        path: /metadata/annotations/cert-manager.io~1cluster-issuer
        value: letsencrypt-staging
      - op: replace
        path: /spec/tls/0/hosts/0
        value: localhost
      - op: replace
        path: /spec/rules/0/host
        value: localhost
```

**Key Design Choices:**

- Dev overlay uses `latest` tag (sandbox uses specific version)
- Localhost hostname for local testing
- Staging Let's Encrypt certificate

#### `Dockerfile.prod` (Root Multi-Stage Build)

```dockerfile
# Multi-stage Dockerfile for Balancer Application
# Produces a single image with Django backend serving the React frontend

# Stage 1: Build Frontend
FROM node:18 AS frontend-builder

WORKDIR /frontend

# Copy frontend package files
COPY frontend/package*.json ./

# Install dependencies
RUN npm ci --legacy-peer-deps

# Copy frontend source
COPY frontend/ ./

# Build frontend - outputs to dist/ as configured in vite.config.ts
RUN npm run build

# Stage 2: Build Backend
FROM python:3.11.4-slim-bullseye

# Set work directory
WORKDIR /usr/src/app

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Install system dependencies
RUN apt-get update && apt-get install -y netcat && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
RUN pip install --upgrade pip
COPY server/requirements.txt .
RUN pip install -r requirements.txt

# Copy backend application code
COPY server/ .

# Copy frontend build from frontend-builder stage to where Django expects it
COPY --from=frontend-builder /frontend/dist ./build

# Run Django collectstatic to gather all static files
RUN python manage.py collectstatic --noinput

# Correct line endings in entrypoint and make executable
RUN sed -i 's/\r$//' entrypoint.prod.sh && chmod +x entrypoint.prod.sh

# Expose port
EXPOSE 8000

# Run entrypoint
ENTRYPOINT ["./entrypoint.prod.sh"]

# Start gunicorn
CMD ["gunicorn", "balancer_backend.wsgi:application", "--bind", "0.0.0.0:8000"]
```

**Key Design Choices:**

- Multi-stage build: frontend + backend in one image
- Frontend built with npm, copied to Django's expected location
- Uses gunicorn for production (not Django runserver)
- Single entrypoint runs migrations and starts server

#### `.github/workflows/containers-publish.yml`

```yaml
name: "Containers: Publish"

on:
  release:
    types: [published]

permissions:
  packages: write

jobs:
  release-containers:
    name: Build and Push
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - name: Login to ghcr.io Docker registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.repository_owner }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Compute Docker container image addresses
        run: |
          DOCKER_REPOSITORY="ghcr.io/${GITHUB_REPOSITORY,,}"
          DOCKER_TAG="${GITHUB_REF:11}"

          echo "DOCKER_REPOSITORY=${DOCKER_REPOSITORY}" >> $GITHUB_ENV
          echo "DOCKER_TAG=${DOCKER_TAG}" >> $GITHUB_ENV

          echo "Using: ${DOCKER_REPOSITORY}/app:${DOCKER_TAG}"

      - name: "Pull previous Docker container image: app:latest"
        run: docker pull "${DOCKER_REPOSITORY}/app:latest" || true

      - name: "Build Docker container image: app:latest"
        run: |
          docker build \
              --cache-from "${DOCKER_REPOSITORY}/app:latest" \
              --file Dockerfile.prod \
              --tag "${DOCKER_REPOSITORY}/app:latest" \
              --tag "${DOCKER_REPOSITORY}/app:${DOCKER_TAG}" \
              .

      - name: "Push Docker container image app:latest"
        run: docker push "${DOCKER_REPOSITORY}/app:latest"

      - name: "Push Docker container image app:v*"
        run: docker push "${DOCKER_REPOSITORY}/app:${DOCKER_TAG}"
```

**Key Design Choices:**

- Triggers on GitHub releases only
- Uses automatic GITHUB_TOKEN (no manual secrets needed)
- Builds from root Dockerfile.prod
- Tags with both `latest` and version tag
- Uses layer caching for faster builds

### Cluster Repository Files

#### `.holo/sources/balancer.toml`

```toml
[holosource]
url = "https://github.com/CodeForPhilly/balancer-main.git"
ref = "refs/heads/main"
```

#### `.holo/lenses/balancer.toml`

```toml
[hololens]
container = "ghcr.io/hologit/lenses/kustomize:latest"

[hololens.input]
root = "balancer"
files = "**"

[hololens.output]
merge = "replace"
```

#### `.holo/branches/k8s-manifests/balancer/manifests.toml`

```toml
[holomapping]
holosource = "balancer"
root = "deploy/manifests/balancer/base"
files = "**"
```

#### `balancer/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: dev

resources:
  - manifests/namespace.yaml
  - manifests/deployment.yaml
  - manifests/service.yaml
  - manifests/ingress.yaml

images:
  - name: ghcr.io/codeforphilly/balancer-main/app
    newTag: "1.0.2"

patches:
  - target:
      kind: Ingress
      name: balancer
    patch: |-
      - op: add
        path: /metadata/annotations/cert-manager.io~1cluster-issuer
        value: letsencrypt-prod
      - op: replace
        path: /spec/tls/0/hosts/0
        value: balancer.sandbox.k8s.phl.io
      - op: replace
        path: /spec/rules/0/host
        value: balancer.sandbox.k8s.phl.io
  - target:
      kind: Namespace
      name: balancer
    patch: |-
      - op: replace
        path: /metadata/name
        value: dev
```

**Key Design Choices:**

- Deploys to `dev` namespace (not `balancer`)
- Pins to specific version: `1.0.2`
- Production Let's Encrypt certificate
- Hostname: balancer.sandbox.k8s.phl.io
- References `manifests/` directory (created by Hologit)

#### `balancer.secrets/balancer-config.yaml`

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: balancer-config
  namespace: balancer
spec:
  encryptedData:
    DATABASE: AgA1uzQmD1wAUs3Q6dgF6ZeOB7avHJtmbAHXFQ8WLBxdgXqmlUAX...
    DEBUG: AgDP6pnrOzd2pdVSnTxH4sDFPuYs8bOBGbvd4XvQ7do2mvjpWHm...
    DJANGO_ALLOWED_HOSTS: AgBKx2rNvYwPzL9mQ5eH8fJ3vR4kS6tU9wX0yA1bC2dE3fG...
    OPENAI_API_KEY: AgCx6udJDYyoy3TvZ8wA9bB0cC1dD2eE3fF4gG5hH6iI7jJ8k...
    PINECONE_API_KEY: AgA0O7RO0sYIo/Kk89DYh3mN6pQ9rS2tU5vW8xY1zA4bC7dE...
    SECRET_KEY: AgBS7yGhT3jK8mN0qS9vW2xZ5aC8dF1gI4jL7oP0qR3tU6vX...
    SQL_DATABASE: AgCk8Qmv5PnX0qR3sT6uV9wY2zA5bC8dE1fG4hI7jK0mN3oP...
    SQL_ENGINE: AgDp3XnzY2aB5cD8eF1gH4iJ7kL0mN3oP6qR9sT2uV5wX8yZ...
    SQL_HOST: AgAr4StV7wX0yZ3aB6cD9eF2gH5iJ8kL1mN4oP7qR0sT3uV6w...
    SQL_PASSWORD: AgBs5TuW8xZ1aB4cD7eF0gH3iJ6kL9mN2oP5qR8sT1uV4wX7y...
    SQL_PORT: AgCt6UvX9yZ2aB5cD8eF1gH4iJ7kL0mN3oP6qR9sT2uV5wX8y...
    SQL_USER: AgDu7VwY0zA3aB6cD9eF2gH5iJ8kL1mN4oP7qR0sT3uV6wX9y...
    # ... additional encrypted keys
```

**Note**: This is the encrypted version, safe to commit to git. Only the cluster's sealed-secrets controller can decrypt these values.

## Deployment Process

### Initial Deployment Timeline

1. **November 15, 2025**: Set up project repository structure
2. **November 16, 2025**: Created Dockerfile.prod with multi-stage build
3. **November 16, 2025**: Configured GitHub Actions workflow
4. **November 16, 2025**: Set up Hologit configuration in cluster repo
5. **November 16, 2025**: Created cluster overlay and sealed secrets
6. **November 16, 2025**: First successful deployment to sandbox

### Deployment Steps (Actual Commands Run)

#### Step 1: Project Repository Setup

```bash
cd /Users/chris/Repositories/balancer-main

# Create directory structure
mkdir -p deploy/manifests/balancer/base
mkdir -p deploy/manifests/balancer/overlays/dev

# Create manifests (files shown above)
# ... created namespace.yaml, deployment.yaml, etc.

# Create root Dockerfile.prod
# ... (multi-stage build shown above)

# Update GitHub Actions
# ... containers-publish.yml

# Commit and push
git add deploy/ Dockerfile.prod .github/
git commit -m "feat(deploy): add Kubernetes manifests and multi-stage Dockerfile"
git push origin main
```

#### Step 2: Create GitHub Release

```bash
# Tag and push
git tag v1.0.2
git push origin v1.0.2

# Or create release via GitHub UI
# This triggers the containers-publish workflow
```

#### Step 3: Cluster Repository Setup

```bash
cd /Users/chris/Repositories/cfp-sandbox-cluster

# Create Hologit configuration
mkdir -p .holo/sources
mkdir -p .holo/lenses
mkdir -p .holo/branches/k8s-manifests/balancer

# Create source definition
cat > .holo/sources/balancer.toml <<EOF
[holosource]
url = "https://github.com/CodeForPhilly/balancer-main.git"
ref = "refs/heads/main"
EOF

# Create lens definition
cat > .holo/lenses/balancer.toml <<EOF
[hololens]
container = "ghcr.io/hologit/lenses/kustomize:latest"

[hololens.input]
root = "balancer"
files = "**"

[hololens.output]
merge = "replace"
EOF

# Create mapping
cat > .holo/branches/k8s-manifests/balancer/manifests.toml <<EOF
[holomapping]
holosource = "balancer"
root = "deploy/manifests/balancer/base"
files = "**"
EOF
```

#### Step 4: Create Cluster Overlay

```bash
# Create overlay kustomization
cat > balancer/kustomization.yaml <<EOF
# ... (content shown above)
EOF
```

#### Step 5: Create Sealed Secrets

```bash
# Create plain secret (with actual values)
kubectl create secret generic balancer-config \
  --namespace=balancer \
  --from-literal=DATABASE="postgres" \
  --from-literal=DEBUG="1" \
  --from-literal=SECRET_KEY="..." \
  # ... all other keys
  --dry-run=client -o yaml > /tmp/balancer-secret.yaml

# Encrypt with kubeseal
kubeseal \
  --controller-name=sealed-secrets \
  --controller-namespace=sealed-secrets \
  --format yaml < /tmp/balancer-secret.yaml > balancer.secrets/balancer-config.yaml

# Remove unencrypted version
rm /tmp/balancer-secret.yaml
```

#### Step 6: Deploy

```bash
# Commit and push to cluster repo
git add .holo/ balancer/ balancer.secrets/
git commit -m "feat(balancer): add deployment configuration"
git push origin main

# GitHub Actions automatically runs Hologit projection
# Wait for workflow to complete (~1-2 minutes)
```

#### Step 7: Verify Deployment

```bash
# Check pods
kubectl get pods -n dev | grep balancer
# balancer-7d4c8b5f9d-x8k2p   1/1     Running   0          2m

# Check ingress
kubectl get ingress -n dev balancer
# NAME       CLASS   HOSTS                            ADDRESS        PORTS     AGE
# balancer   nginx   balancer.sandbox.k8s.phl.io     203.0.113.1    80, 443   2m

# Check logs
kubectl logs -n dev deployment/balancer

# Check certificate
kubectl get certificate -n dev
# NAME           READY   SECRET         AGE
# balancer-tls   True    balancer-tls   2m
```

#### Step 8: Access Application

Open browser: <https://balancer.sandbox.k8s.phl.io>

✅ Application is live!

## Architecture Decisions

### Single-Container vs Multi-Container

**Decision**: Single container with Django serving both API and frontend

**Rationale**:

- Simpler deployment (one pod instead of two)
- No need for complex ingress routing rules
- Django's catch-all URL pattern handles SPA routing
- Fewer moving parts = easier to debug
- Lower resource usage

**Trade-offs**:

- Frontend can't scale independently of backend
- Single point of failure (acceptable for sandbox)
- Frontend updates require full image rebuild

**Alternative Considered**: Separate frontend (nginx) and backend (Django) containers with ingress-based routing. Rejected as over-complicated for this use case.

### Multi-Stage Docker Build

**Decision**: Build frontend in Node.js stage, copy to Python backend stage

**Rationale**:

- Self-contained build (no external build scripts)
- Docker-native (follows Docker best practices)
- Reproducible builds
- Proper layer caching

**Previous Approach**: Separate `deployment.sh` script that built frontend before docker build. This violated Docker principles by depending on external build state.

### Kustomize vs Helm

**Decision**: Use Kustomize for manifest management

**Rationale**:

- Simpler for applications without complex configuration
- More transparent (pure YAML, no templating)
- Better for learning Kubernetes
- Easier to diff and review changes
- Native kubectl support

**When to use Helm**: Applications with many configuration options, external charts, or complex dependencies.

### Environment Configuration

**Decision**: All config via environment variables from Kubernetes Secret

**Rationale**:

- 12-factor app principles
- No config baked into image
- Same image works in all environments
- Easy to update config without rebuilding

**Implementation**: Django reads from `os.environ`, frontend env vars baked in at build time.

### Secret Management

**Decision**: Use SealedSecrets for all sensitive data

**Rationale**:

- Safe to commit encrypted secrets to git
- GitOps-friendly
- Automatic decryption in cluster
- No manual secret creation needed

**Alternative Considered**: External secret managers (Vault, AWS Secrets Manager). Rejected as too complex for sandbox environment.

### Namespace Strategy

**Decision**: Deploy to `dev` namespace (not `balancer`)

**Rationale**:

- Sandbox cluster uses `dev` namespace for all projects
- Future: Could have separate namespaces per project
- Allows namespace-level RBAC and resource quotas

**Implementation**: Base manifests define `balancer` namespace, overlay patches to `dev`.

### Image Tagging Strategy

**Decision**: Use semantic version tags (v1.0.2) with latest

**Rationale**:

- Clear versioning for rollbacks
- Latest tag for convenience
- Matches GitHub release tags
- Easy to track what's deployed

**Implementation**: GitHub Actions creates both `latest` and `v{version}` tags on release.

### Ingress Configuration

**Decision**: Single catch-all path routing to service

**Rationale**:

- Django handles all routing internally
- No need to enumerate API paths in ingress
- Simpler ingress configuration
- Easier to add new routes

**Alternative Considered**: Path-based routing with separate paths for API (/api/*) and frontend (/). Rejected as unnecessary complexity since Django handles this.

### Resource Limits

**Decision**: No resource limits initially

**Rationale**:

- Unknown resource requirements initially
- Sandbox environment has capacity
- Can add limits after monitoring usage

**Future**: Should add based on observed usage:

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

## Lessons Learned

### What Worked Well

1. **Multi-stage Dockerfile**: Clean, self-contained builds
2. **Kustomize overlays**: Easy to manage environment differences
3. **Hologit projection**: Automatic manifest sync from source repo
4. **SealedSecrets**: Safe secret management in git
5. **Documentation**: secret.template.yaml helped document requirements

### What Could Be Improved

1. **Initial complexity**: Took time to understand Hologit workflow
2. **Debug cycle**: Long feedback loop (commit → push → wait for projection)
3. **Local testing**: No easy way to test Hologit projection locally
4. **Resource limits**: Should have added from the start
5. **Monitoring**: No proactive alerts for issues

### Common Pitfalls Encountered

1. **Image tag mismatch**: Forgot to update image tag in cluster overlay
2. **Secret keys**: Typo in secret key name caused crash
3. **Namespace confusion**: Base vs overlay namespace caused deployment to wrong namespace
4. **Certificate delay**: cert-manager took ~5 minutes to issue certificate
5. **Django static files**: Initially forgot `collectstatic` in Dockerfile

## Next Steps for Balancer

- [ ] Add resource requests/limits based on monitoring
- [ ] Set up horizontal pod autoscaling
- [ ] Add database backup strategy
- [ ] Implement proper logging aggregation
- [ ] Add monitoring/alerting (Prometheus/Grafana)
- [ ] Document local development setup
- [ ] Create staging environment
- [ ] Set up automated testing before deployment

## References

- **Project Repository**: <https://github.com/CodeForPhilly/balancer-main>
- **Container Images**: <https://github.com/orgs/CodeForPhilly/packages?repo_name=balancer-main>
- **Live Application**: <https://balancer.sandbox.k8s.phl.io>
- **Deployment Guide**: [../project-deployment-guide.md](../project-deployment-guide.md)
