# Quick Reference

Fast reference for common commands and configurations when working with the cfp-sandbox-cluster.

## File Locations

### Project Repository

```
your-project/
├── deploy/manifests/your-project/base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── secret.template.yaml
├── Dockerfile.prod
└── .github/workflows/containers-publish.yml
```

### Cluster Repository

```
cfp-sandbox-cluster/
├── .holo/sources/your-project.toml
├── .holo/lenses/your-project.toml
├── .holo/branches/k8s-manifests/your-project/manifests.toml
├── your-project/kustomization.yaml
└── your-project.secrets/*.yaml
```

## Common kubectl Commands

### Pod Management

```bash
# List pods in namespace
kubectl get pods -n your-project

# Describe pod (detailed info, events)
kubectl describe pod -n your-project {pod-name}

# View logs (live)
kubectl logs -f -n your-project deployment/your-project

# View logs (previous crashed container)
kubectl logs -n your-project {pod-name} --previous

# Execute command in pod
kubectl exec -it -n your-project {pod-name} -- /bin/bash

# Check environment variables
kubectl exec -n your-project {pod-name} -- env

# Delete pod (will be recreated)
kubectl delete pod -n your-project {pod-name}
```

### Deployment Management

```bash
# View deployments
kubectl get deployments -n your-project

# Describe deployment
kubectl describe deployment -n your-project your-project

# Restart deployment (e.g., to pick up new secret)
kubectl rollout restart deployment/your-project -n your-project

# View rollout status
kubectl rollout status deployment/your-project -n your-project

# View rollout history
kubectl rollout history deployment/your-project -n your-project

# Rollback to previous version
kubectl rollout undo deployment/your-project -n your-project

# Rollback to specific revision
kubectl rollout undo deployment/your-project -n your-project --to-revision=2

# Scale deployment
kubectl scale deployment/your-project -n your-project --replicas=2
```

### Service & Ingress

```bash
# View services
kubectl get service -n your-project

# View ingress
kubectl get ingress -n your-project

# Describe ingress
kubectl describe ingress -n your-project your-project

# Check certificate
kubectl get certificate -n your-project
kubectl describe certificate -n your-project your-project-tls

# Port forward to local machine
kubectl port-forward -n your-project service/your-project 8000:8000
# Access at http://localhost:8000
```

### Secret Management

```bash
# List secrets
kubectl get secrets -n your-project

# View secret keys (not values)
kubectl get secret -n your-project your-project-config -o jsonpath='{.data}'

# View secret in YAML
kubectl get secret -n your-project your-project-config -o yaml

# Check SealedSecrets
kubectl get sealedsecrets -n your-project
```

### Debugging

```bash
# View all resources
kubectl get all -n your-project

# View events (sorted by time)
kubectl get events -n your-project --sort-by='.lastTimestamp'

# Get resource usage
kubectl top pods -n your-project

# View node information
kubectl get nodes
kubectl describe node {node-name}
```

## Hologit Configuration Templates

### Source Definition

```toml
# .holo/sources/your-project.toml
[holosource]
url = "https://github.com/CodeForPhilly/your-project.git"
ref = "refs/heads/main"
```

### Lens Definition (Kustomize)

```toml
# .holo/lenses/your-project.toml
[hololens]
container = "ghcr.io/hologit/lenses/kustomize:latest"

[hololens.input]
root = "your-project"
files = "**"

[hololens.output]
merge = "replace"
```

### Mapping Definition

```toml
# .holo/branches/k8s-manifests/your-project/manifests.toml
[holomapping]
holosource = "your-project"
root = "deploy/manifests/your-project/base"
files = "**"
```

## Kustomize Snippets

### Base Kustomization

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

### Cluster Overlay

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
```

## Secret Management

### Create SealedSecret

```bash
# 1. Create plain secret
kubectl create secret generic your-project-config \
  --namespace=your-project \
  --from-literal=KEY1="value1" \
  --from-literal=KEY2="value2" \
  --dry-run=client -o yaml > /tmp/secret.yaml

# 2. Encrypt with kubeseal
kubeseal \
  --controller-name=sealed-secrets \
  --controller-namespace=sealed-secrets \
  --format yaml < /tmp/secret.yaml > your-project.secrets/config.yaml

# 3. Clean up
rm /tmp/secret.yaml

# 4. Commit encrypted secret
git add your-project.secrets/
git commit -m "feat(your-project): add sealed secrets"
```

### Create Secret from File

```bash
kubectl create secret generic your-project-config \
  --namespace=your-project \
  --from-file=config.json=./config.json \
  --dry-run=client -o yaml | \
kubeseal --format yaml > your-project.secrets/config.yaml
```

### Create Secret from Env File

```bash
kubectl create secret generic your-project-config \
  --namespace=your-project \
  --from-env-file=.env \
  --dry-run=client -o yaml | \
kubeseal --format yaml > your-project.secrets/config.yaml
```

## Container Publishing

### Tag and Release

```bash
# In project repository
git tag v1.0.0
git push origin v1.0.0

# Or create release via GitHub UI
# This triggers containers-publish workflow
```

### Build and Test Locally

```bash
# Build
docker build -f Dockerfile.prod -t your-app:test .

# Run with env file
docker run -p 8000:8000 --env-file .env your-app:test

# Run with individual env vars
docker run -p 8000:8000 -e KEY=value your-app:test

# Test
curl http://localhost:8000
```

## Version Updates

### Update to New Version

```bash
# 1. In project repo: create release (triggers build)
git tag v1.1.0
git push origin v1.1.0

# 2. In cluster repo: update image tag
cd cfp-sandbox-cluster
vim your-project/kustomization.yaml
# Change newTag: "1.0.0" to "1.1.0"

# 3. Commit and push
git add your-project/kustomization.yaml
git commit -m "feat(your-project): update to v1.1.0"
git push origin main

# 4. Verify
kubectl get pods -n your-project
kubectl describe pod -n your-project {pod-name} | grep Image
```

## Troubleshooting Quick Checks

### Pod Won't Start

```bash
# 1. Check status
kubectl get pods -n your-project

# 2. View events
kubectl describe pod -n your-project {pod-name}

# 3. Check logs
kubectl logs -n your-project {pod-name}
kubectl logs -n your-project {pod-name} --previous

# 4. Common fixes
# - ImagePullBackOff: Check image name and tag
# - CrashLoopBackOff: Check logs for application errors
# - Pending: Check events for scheduling issues
```

### Ingress Not Working

```bash
# 1. Check ingress
kubectl get ingress -n your-project
kubectl describe ingress -n your-project your-project

# 2. Check certificate
kubectl get certificate -n your-project
kubectl describe certificate -n your-project your-project-tls

# 3. Check service
kubectl get service -n your-project
kubectl describe service -n your-project your-project

# 4. Test service directly (bypass ingress)
kubectl port-forward -n your-project service/your-project 8000:8000
# Access http://localhost:8000
```

### Secret Not Found

```bash
# 1. Check secret exists
kubectl get secret -n your-project

# 2. Check SealedSecret
kubectl get sealedsecret -n your-project

# 3. View sealed-secrets controller logs
kubectl logs -n sealed-secrets deployment/sealed-secrets-controller

# 4. Recreate if needed
# Delete and recreate the SealedSecret, then commit/push
```

## Common Patterns

### Namespace Pattern

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: your-project
```

### Deployment Pattern

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: your-project
  labels:
    app: your-project
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
        - name: app
          image: ghcr.io/codeforphilly/your-project/app
          envFrom:
            - secretRef:
                name: your-project-config
          ports:
            - containerPort: 8000
```

### Service Pattern

```yaml
apiVersion: v1
kind: Service
metadata:
  name: your-project
  labels:
    app: your-project
spec:
  type: ClusterIP
  ports:
    - port: 8000
      targetPort: 8000
      name: http
  selector:
    app: your-project
```

### Ingress Pattern

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: your-project
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - your-project.sandbox.k8s.phl.io
      secretName: your-project-tls
  rules:
    - host: your-project.sandbox.k8s.phl.io
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

## Naming Conventions

### Container Images

- Registry: `ghcr.io`
- Format: `ghcr.io/codeforphilly/{repo-name}/app`
- Tags: `latest`, `v1.0.0`, `v1.0.1`, etc.

### Kubernetes Resources

- Namespace: `your-project`
- Deployment: `your-project`
- Service: `your-project`
- Ingress: `your-project`
- Secret: `your-project-config`
- TLS Secret: `your-project-tls` (auto-created)

### Hostnames

- Pattern: `{project}.sandbox.k8s.phl.io`
- Example: `balancer.sandbox.k8s.phl.io`

### Git Conventions

- Commit format: `feat(your-project): description`
- Version tags: `v1.0.0`, `v1.1.0`, etc. (semantic versioning)

## URLs and Resources

- **Cluster Domain**: `*.sandbox.k8s.phl.io`
- **Container Registry**: <https://github.com/orgs/CodeForPhilly/packages>
- **Cluster Repository**: <https://github.com/CodeForPhilly/cfp-sandbox-cluster>
- **Documentation Site**: <https://codeforphilly.github.io/cfp-sandbox-cluster>
- **Kustomize Docs**: <https://kustomize.io>
- **kubectl Cheat Sheet**: <https://kubernetes.io/docs/reference/kubectl/cheatsheet/>
- **SealedSecrets**: <https://github.com/bitnami-labs/sealed-secrets>

## Getting Help

- **Slack**: #civic-tech-infrastructure
- **GitHub Issues**: <https://github.com/CodeForPhilly/cfp-sandbox-cluster/issues>
- **Full Documentation**: [project-deployment-guide.md](project-deployment-guide.md)
- **Example Walkthrough**: [examples/balancer-walkthrough.md](examples/balancer-walkthrough.md)
