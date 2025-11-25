# Project Deployment Guide

This guide walks you through deploying your application to the Code for Philly sandbox Kubernetes cluster.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Step-by-Step Deployment](#step-by-step-deployment)
- [Container Publishing Setup](#container-publishing-setup)
- [Secret Management](#secret-management)
- [Version Updates](#version-updates)
- [Troubleshooting](#troubleshooting)

## Prerequisites

### Tools Required

1. **kubectl** - Kubernetes command-line tool

   ```bash
   # Install via homebrew (macOS)
   brew install kubectl

   # Or download from kubernetes.io
   ```

2. **git** - Version control

   ```bash
   brew install git
   ```

3. **kubeseal** - For encrypting secrets (will be covered later)

   ```bash
   brew install kubeseal
   ```

### Access Requirements

1. **GitHub Organization Access**
   - Member of CodeForPhilly GitHub organization
   - Access to your project repository
   - Access to `cfp-sandbox-cluster` repository

2. **Container Registry**
   - GitHub Container Registry (ghcr.io) access
   - Enabled via GitHub Actions in your repository

3. **Kubernetes Cluster Access**
   - kubectl configured with cluster credentials
   - Contact cluster administrators for access

### Repository Setup

Your project should be set up as a GitHub repository under the CodeForPhilly organization (e.g., `CodeForPhilly/your-project`).

## Architecture Overview

The cfp-sandbox-cluster uses a GitOps workflow powered by **Hologit** to manage deployments.

### How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. PROJECT REPOSITORY (e.g., balancer-main)                    │
│    deploy/manifests/{project}/base/                             │
│    └── Kubernetes manifests (deployment, service, ingress)     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ GitHub Release
                              ↓
                    ┌──────────────────┐
                    │  Container Image │
                    │  ghcr.io/.../app │
                    └──────────────────┘
                              │
            Hologit Projects  │
                ↓             │
┌─────────────────────────────────────────────────────────────────┐
│ 2. CLUSTER REPOSITORY (cfp-sandbox-cluster)                    │
│    .holo/                      {project}/                       │
│    ├── sources/{project}.toml  └── kustomization.yaml          │
│    ├── lenses/{project}.toml       (environment-specific)      │
│    └── branches/.../mapping    {project}.secrets/              │
│                                └── SealedSecret manifests       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ GitOps
                              ↓
                    ┌──────────────────┐
                    │ Kubernetes       │
                    │ Cluster          │
                    │ *.sandbox.k8s.   │
                    │   phl.io         │
                    └──────────────────┘
```

### Key Concepts

1. **Hologit Projection**: Your manifests are "projected" from your project repository into the cluster repository
2. **Environment Overlays**: Cluster-specific configuration (hostname, namespace, image tags) is applied via Kustomize overlays
3. **Sealed Secrets**: Secrets are encrypted and stored safely in git
4. **GitOps Automation**: Changes to the cluster repository trigger automated deployments

## Step-by-Step Deployment

This guide uses the Kustomize approach, which is recommended for new projects. We'll use the **balancer** project as our example throughout.

### Step 1: Prepare Your Project Repository

Create the following directory structure in your project repository:

```
your-project/
├── deploy/
│   └── manifests/
│       └── your-project/
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
└── Dockerfile.prod
```

**Why this structure?**

- `base/` contains environment-agnostic Kubernetes manifests
- `overlays/` contains environment-specific overrides (dev, staging, prod)
- All manifests live in your project repo (single source of truth)

### Step 2: Create Base Kubernetes Manifests

#### 2.1 Namespace (`namespace.yaml`)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: your-project
```

#### 2.2 Deployment (`deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: your-project
  name: your-project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: your-project
  template:
    metadata:
      labels:
        app: your-project
    spec:
      containers:
        - image: ghcr.io/codeforphilly/your-project/app
          name: app
          envFrom:
            - secretRef:
                name: your-project-config
          ports:
            - containerPort: 8000
          readinessProbe:
            httpGet:
              path: /
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
```

**Key Points:**

- Use `envFrom` to load all config from a Secret
- Image name should match your ghcr.io registry
- Readiness probe ensures traffic only goes to healthy pods
- No tag specified (added by overlay)

#### 2.3 Service (`service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: your-project
  labels:
    app: your-project
spec:
  ports:
    - name: http
      port: 8000
      targetPort: 8000
  selector:
    app: your-project
```

**Key Points:**

- ClusterIP service (internal only)
- Port 8000 is common for web apps, adjust as needed
- Selector must match deployment labels

#### 2.4 Ingress (`ingress.yaml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: your-project
  annotations: {}
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - HOSTNAME_PLACEHOLDER
      secretName: your-project-tls
  rules:
    - host: HOSTNAME_PLACEHOLDER
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: your-project
                port:
                  number: 8000
```

**Key Points:**

- Use `HOSTNAME_PLACEHOLDER` - will be patched by overlay
- Empty annotations - will be added by overlay
- TLS secret name follows pattern: `{project}-tls`
- cert-manager will automatically provision SSL certificate

#### 2.5 Kustomization (`kustomization.yaml`)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

#### 2.6 Secret Template (`secret.template.yaml`)

```yaml
# Secret Template for Your Project
#
# IMPORTANT: This file is a TEMPLATE only. Do NOT create actual Secret
# manifests in this repository. Secrets should be created in the cluster
# repository using SealedSecrets.
#
# To use: Copy this template to the cluster repo and encrypt with kubeseal
#
# Required secret keys for your application:
# -------------------
# DATABASE_URL: "postgresql://user:pass@host:5432/dbname"
# SECRET_KEY: "your-secret-key-here"
# API_KEY: "your-api-key-here"
# ... (list all required environment variables)
```

**Key Points:**

- This is documentation only
- Lists all required secret keys
- Actual secrets created in cluster repo (Step 6)

### Step 3: Create Dev Overlay

Create `deploy/manifests/your-project/overlays/dev/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: your-project

resources:
  - ../../base

images:
  - name: ghcr.io/codeforphilly/your-project/app
    newTag: latest

patches:
  - target:
      kind: Ingress
      name: your-project
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

**Key Points:**

- Sets namespace
- Adds image tag (latest for dev)
- Patches ingress for local testing

### Step 4: Set Up Container Publishing

Create `.github/workflows/containers-publish.yml` in your project repository:

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

**Key Points:**

- Triggers on GitHub releases
- Uses GITHUB_TOKEN (automatically provided)
- Builds and pushes to ghcr.io
- Tags with both `latest` and version tag

**To trigger a build:**

```bash
git tag v1.0.0
git push origin v1.0.0
# Or create a release via GitHub UI
```

### Step 5: Configure Hologit in Cluster Repository

Now switch to the `cfp-sandbox-cluster` repository. You'll need to create three configuration files:

#### 5.1 Source Definition

Create `.holo/sources/your-project.toml`:

```toml
[holosource]
url = "https://github.com/CodeForPhilly/your-project.git"
ref = "refs/heads/main"
```

**Key Points:**

- `url`: Your project's GitHub URL
- `ref`: Branch to pull from (usually `main`)

#### 5.2 Lens Definition

Create `.holo/lenses/your-project.toml`:

```toml
[hololens]
container = "ghcr.io/hologit/lenses/kustomize:latest"

[hololens.input]
root = "your-project"
files = "**"

[hololens.output]
merge = "replace"
```

**Key Points:**

- Uses Kustomize lens
- `root`: Must match your holosource name
- `merge = "replace"`: Overwrites existing files on each projection

#### 5.3 Mapping Definition

Create `.holo/branches/k8s-manifests/your-project/manifests.toml`:

```toml
[holomapping]
holosource = "your-project"
root = "deploy/manifests/your-project/base"
files = "**"
```

**Key Points:**

- Maps to your base manifests directory
- Files will be projected to `your-project/manifests/` in cluster repo

### Step 6: Create Cluster Overlay

Create `your-project/kustomization.yaml` in the cluster repository:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: your-project

resources:
  - manifests/namespace.yaml
  - manifests/deployment.yaml
  - manifests/service.yaml
  - manifests/ingress.yaml

images:
  - name: ghcr.io/codeforphilly/your-project/app
    newTag: "1.0.0"

patches:
  - target:
      kind: Ingress
      name: your-project
    patch: |-
      - op: add
        path: /metadata/annotations/cert-manager.io~1cluster-issuer
        value: letsencrypt-prod
      - op: replace
        path: /spec/tls/0/hosts/0
        value: your-project.sandbox.k8s.phl.io
      - op: replace
        path: /spec/rules/0/host
        value: your-project.sandbox.k8s.phl.io
  - target:
      kind: Namespace
      name: your-project
    patch: |-
      - op: replace
        path: /metadata/name
        value: your-project
```

**Key Points:**

- References `manifests/` directory (created by Hologit projection)
- Sets specific image version
- Patches ingress with production hostname
- Sets cert-manager to use production Let's Encrypt

### Step 7: Manage Secrets

#### 7.1 Create Plain Secret Locally

First, create a plain Kubernetes Secret with your actual values:

```bash
kubectl create secret generic your-project-config \
  --namespace=your-project \
  --from-literal=DATABASE_URL="postgresql://..." \
  --from-literal=SECRET_KEY="your-secret-key" \
  --from-literal=API_KEY="your-api-key" \
  --dry-run=client -o yaml > secret.yaml
```

#### 7.2 Encrypt with kubeseal

```bash
kubeseal \
  --controller-name=sealed-secrets \
  --controller-namespace=sealed-secrets \
  --format yaml < secret.yaml > sealed-secret.yaml
```

#### 7.3 Store in Cluster Repository

Create directory and move sealed secret:

```bash
mkdir -p your-project.secrets
mv sealed-secret.yaml your-project.secrets/your-project-config.yaml
rm secret.yaml  # Delete unencrypted version!
```

**Your sealed secret will look like:**

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: your-project-config
  namespace: your-project
spec:
  encryptedData:
    DATABASE_URL: AgBS7yGh...encrypted...
    SECRET_KEY: AgCk8Qmv...encrypted...
    API_KEY: AgDp3Xnz...encrypted...
```

**Key Points:**

- Encrypted values are safe to commit to git
- Only the cluster's sealed-secrets controller can decrypt
- Must re-seal if you change the secret

### Step 8: Deploy

#### 8.1 Commit and Push

In the `cfp-sandbox-cluster` repository:

```bash
git add .holo/ your-project/ your-project.secrets/
git commit -m "feat(your-project): add deployment configuration"
git push origin main
```

#### 8.2 Trigger Hologit Projection

The GitHub Actions workflow will automatically:

1. Run Hologit to project your manifests
2. Commit to `releases/k8s-manifests` branch
3. Trigger deployment to the cluster

#### 8.3 Verify Deployment

Check that your pods are running:

```bash
kubectl get pods -n your-project

# Expected output:
NAME                            READY   STATUS    RESTARTS   AGE
your-project-7d4c8b5f9d-x8k2p   1/1     Running   0          2m
```

Check the ingress:

```bash
kubectl get ingress -n your-project

# Expected output:
NAME           CLASS   HOSTS                              ADDRESS        PORTS     AGE
your-project   nginx   your-project.sandbox.k8s.phl.io   203.0.113.1    80, 443   2m
```

View logs:

```bash
kubectl logs -n your-project deployment/your-project
```

#### 8.4 Access Your Application

Navigate to: `https://your-project.sandbox.k8s.phl.io`

Your application should be live! 🎉

## Container Publishing Setup

### Multi-Stage Dockerfile Example

If your application has both frontend and backend, use a multi-stage build:

```dockerfile
# Stage 1: Build Frontend
FROM node:18 AS frontend-builder

WORKDIR /frontend
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ ./
RUN npm run build

# Stage 2: Build Backend
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
RUN apt-get update && apt-get install -y netcat && rm -rf /var/lib/apt/lists/*
COPY backend/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY backend/ .

# Copy frontend build from stage 1
COPY --from=frontend-builder /frontend/dist ./static

EXPOSE 8000

CMD ["gunicorn", "app:application", "--bind", "0.0.0.0:8000"]
```

### Image Naming Convention

- Repository: `ghcr.io/codeforphilly/{repo-name}`
- Image: `ghcr.io/codeforphilly/{repo-name}/app`
- Tags: Version tags (e.g., `v1.0.0`) and `latest`

### Testing Container Locally

```bash
# Build
docker build -f Dockerfile.prod -t your-app:test .

# Run
docker run -p 8000:8000 --env-file .env your-app:test

# Test
curl http://localhost:8000
```

## Secret Management

### Best Practices

1. **Never commit unencrypted secrets** to any git repository
2. **Use SealedSecrets** for all sensitive configuration
3. **Document required secrets** in `secret.template.yaml`
4. **Rotate secrets periodically** by creating new sealed secrets
5. **Use separate secrets** for different concerns (database, API keys, etc.)

### Creating Multiple Secrets

If you have multiple secrets (e.g., database, external APIs):

```bash
# Database secret
kubectl create secret generic your-project-db \
  --namespace=your-project \
  --from-literal=DATABASE_URL="..." \
  --dry-run=client -o yaml | \
kubeseal --format yaml > your-project.secrets/database.yaml

# API keys secret
kubectl create secret generic your-project-api \
  --namespace=your-project \
  --from-literal=OPENAI_API_KEY="..." \
  --from-literal=STRIPE_API_KEY="..." \
  --dry-run=client -o yaml | \
kubeseal --format yaml > your-project.secrets/api-keys.yaml
```

Reference in deployment:

```yaml
envFrom:
  - secretRef:
      name: your-project-db
  - secretRef:
      name: your-project-api
```

### Updating Secrets

To update a secret:

1. Create new plain secret with updated values
2. Seal with kubeseal
3. Replace the sealed secret file in `your-project.secrets/`
4. Commit and push
5. Restart deployment to pick up new secret:

   ```bash
   kubectl rollout restart deployment/your-project -n your-project
   ```

## Version Updates

### Releasing a New Version

1. **In your project repository:**

   ```bash
   # Create and push a tag
   git tag v1.1.0
   git push origin v1.1.0

   # Or create a GitHub Release via the UI
   ```

2. **In the cluster repository:**

   Update `your-project/kustomization.yaml`:

   ```yaml
   images:
     - name: ghcr.io/codeforphilly/your-project/app
       newTag: "1.1.0"  # Changed from 1.0.0
   ```

   Commit and push:

   ```bash
   git add your-project/kustomization.yaml
   git commit -m "feat(your-project): update to v1.1.0"
   git push origin main
   ```

3. **Verify deployment:**

   ```bash
   kubectl get pods -n your-project
   kubectl describe pod -n your-project {pod-name}
   # Check that image is ghcr.io/.../app:1.1.0
   ```

### Rollback to Previous Version

If something goes wrong:

```bash
# Revert the version in kustomization.yaml
git revert HEAD
git push origin main

# Or manually edit and set to previous version
```

### Deployment History

View deployment history:

```bash
kubectl rollout history deployment/your-project -n your-project

# Rollback to previous revision
kubectl rollout undo deployment/your-project -n your-project

# Rollback to specific revision
kubectl rollout undo deployment/your-project -n your-project --to-revision=2
```

## Troubleshooting

### Pod Not Starting

**Check pod status:**

```bash
kubectl get pods -n your-project
```

**Common statuses:**

- `ImagePullBackOff`: Image doesn't exist or can't be pulled
- `CrashLoopBackOff`: Container starts but immediately crashes
- `Pending`: Waiting for resources or scheduling

**View detailed information:**

```bash
kubectl describe pod -n your-project {pod-name}
```

**View container logs:**

```bash
kubectl logs -n your-project {pod-name}

# For previous crashed container
kubectl logs -n your-project {pod-name} --previous
```

### Image Pull Errors

**Error: `ImagePullBackOff`**

Check that:

1. Image name is correct in deployment
2. Tag exists in ghcr.io
3. Image is public or cluster has pull credentials

View specific error:

```bash
kubectl describe pod -n your-project {pod-name} | grep -A 10 "Events:"
```

### Application Errors

**View live logs:**

```bash
kubectl logs -f -n your-project deployment/your-project
```

**Execute commands in running pod:**

```bash
kubectl exec -it -n your-project {pod-name} -- /bin/bash

# Check environment variables
kubectl exec -n your-project {pod-name} -- env
```

### Ingress Not Working

**Check ingress status:**

```bash
kubectl get ingress -n your-project
kubectl describe ingress -n your-project your-project
```

**Common issues:**

- Certificate not issued: Check cert-manager logs
- DNS not resolving: Check hostname configuration
- Backend service not found: Check service name in ingress matches actual service

**Check certificate:**

```bash
kubectl get certificate -n your-project
kubectl describe certificate -n your-project your-project-tls
```

### Secret Not Found

**Error: `couldn't find key ... in Secret`**

Check that secret exists:

```bash
kubectl get secret -n your-project your-project-config
```

View secret keys (not values):

```bash
kubectl get secret -n your-project your-project-config -o jsonpath='{.data}'
```

If secret is missing, ensure:

1. SealedSecret is in `your-project.secrets/`
2. SealedSecret was committed and pushed
3. sealed-secrets controller has processed it

### Common kubectl Commands

```bash
# View all resources in namespace
kubectl get all -n your-project

# Restart deployment
kubectl rollout restart deployment/your-project -n your-project

# Scale deployment
kubectl scale deployment/your-project -n your-project --replicas=2

# View events
kubectl get events -n your-project --sort-by='.lastTimestamp'

# Delete pod (will be recreated automatically)
kubectl delete pod -n your-project {pod-name}

# Port forward to local machine
kubectl port-forward -n your-project service/your-project 8000:8000
# Then access at http://localhost:8000
```

### Getting Help

If you're stuck:

1. **Check GitHub Actions**: Look at the build-k8s-manifests workflow in cfp-sandbox-cluster
2. **Review Examples**: Look at other projects like balancer, choose-native-plants
3. **Ask for Help**: Reach out in the Code for Philly Slack (#civic-tech-infrastructure)

## Next Steps

- **See a complete example**: Check out [balancer-walkthrough.md](examples/balancer-walkthrough.md)
- **Quick commands**: Reference [quick-reference.md](quick-reference.md)
- **Helm deployment**: For Helm-based projects (coming soon)

## Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kustomize Documentation](https://kustomize.io/)
- [SealedSecrets](https://github.com/bitnami-labs/sealed-secrets)
- [Hologit](https://github.com/JarvusInnovations/hologit)
