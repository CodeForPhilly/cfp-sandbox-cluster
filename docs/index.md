# Philly (sandbox) Civic Cloud Documentation

Welcome to the documentation for Code for Philly's sandbox Kubernetes cluster. This cluster provides a shared platform for civic tech projects to deploy and host their applications.

## 🚀 Getting Started

New to deploying on the sandbox cluster? Start here:

1. **[Project Deployment Guide](project-deployment-guide.md)** - Complete guide to deploying your application
2. **[Balancer Walkthrough](examples/balancer-walkthrough.md)** - Real-world example with actual code
3. **[Quick Reference](quick-reference.md)** - Command reference and common patterns

## 📚 Documentation

### For Project Teams

- **[Project Deployment Guide](project-deployment-guide.md)** - End-to-end deployment instructions
    - Prerequisites and access requirements
    - Step-by-step deployment process
    - Container publishing setup
    - Secret management with SealedSecrets
    - Version updates and rollbacks
    - Troubleshooting common issues

- **[Quick Reference](quick-reference.md)** - Fast reference for common tasks
    - kubectl command cheat sheet
    - Hologit configuration templates
    - Kustomize patterns
    - Troubleshooting quick checks

### Examples

- **[Balancer Deployment Walkthrough](examples/balancer-walkthrough.md)** - Complete example
    - Full file listings with explanations
    - Architecture decisions and rationale
    - Real deployment process with actual commands
    - Lessons learned and best practices

### Cluster Services

- **[Data Warehouse](data-warehouse/README.md)** - Shared PostgreSQL database and Metabase
    - [Add User Guide](data-warehouse/add-user.md)
    - [Initial Setup](data-warehouse/initial-setup.md)

- **[Echo Service](echo-service.md)** - Simple connectivity verification service

## 🏗️ Cluster Architecture

The cfp-sandbox-cluster uses a GitOps workflow powered by **Hologit** to manage deployments.

### Key Components

- **Kubernetes Cluster**: Hosts all project deployments
- **Hologit**: Projects manifests from project repos into cluster repo
- **Kustomize**: Manages environment-specific configuration
- **SealedSecrets**: Encrypts secrets safely in git
- **cert-manager**: Automatic TLS certificates via Let's Encrypt
- **nginx-ingress**: Routes traffic to services

### Deployment Flow

```
Project Repository
    ↓ (Hologit projects manifests)
Cluster Repository
    ↓ (GitOps automation)
Kubernetes Cluster
    ↓
https://*.sandbox.k8s.phl.io
```

## 🔑 Key Concepts

### Hologit Projection

Your Kubernetes manifests live in your project repository but are automatically "projected" into the cluster repository. This allows:

- Single source of truth in your project repo
- Automatic updates when you push changes
- Environment-specific overlays in cluster repo

### Kustomize Overlays

Base manifests define your application. Overlays patch those manifests for specific environments:

- **Base** (in project repo): Environment-agnostic configuration
- **Overlay** (in cluster repo): Sandbox-specific patches (hostname, secrets, image tags)

### SealedSecrets

Sensitive configuration is encrypted before being committed to git. Only the cluster can decrypt these secrets.

## 🌐 Cluster Information

- **Domain**: `*.sandbox.k8s.phl.io`
- **Container Registry**: GitHub Container Registry (ghcr.io)
- **SSL Certificates**: Automatic via Let's Encrypt
- **Ingress Controller**: nginx
- **GitOps**: Automatic deployment on push to main

## 📦 Currently Deployed Projects

Projects currently running on the sandbox cluster:

- **balancer** - Medication management application ([balancer.sandbox.k8s.phl.io](https://balancer.sandbox.k8s.phl.io))
- **choose-native-plants** - PA wildflower selector
- **prevention-point** - Harm reduction services
- **metabase** - Data visualization and analytics
- **data-warehouse** - Shared PostgreSQL database

## 🛠️ Prerequisites for Deployment

To deploy a project to the sandbox cluster, you'll need:

1. **Access**: Member of CodeForPhilly GitHub organization
2. **Tools**: kubectl, git, kubeseal
3. **Repository**: Project repository under CodeForPhilly org
4. **Container**: Docker image published to ghcr.io
5. **Manifests**: Kubernetes manifests in your project repo

See the [Project Deployment Guide](project-deployment-guide.md) for detailed prerequisites.

## 🆘 Getting Help

### Documentation

- Start with the [Project Deployment Guide](project-deployment-guide.md)
- Check the [Quick Reference](quick-reference.md) for commands
- Review the [Balancer example](examples/balancer-walkthrough.md) for patterns

### Community Support

- **Slack**: #civic-tech-infrastructure channel
- **GitHub Issues**: [cfp-sandbox-cluster issues](https://github.com/CodeForPhilly/cfp-sandbox-cluster/issues)
- **Hack Nights**: Tuesday evenings at Code for Philly

## 📖 Additional Resources

### External Documentation

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kustomize Documentation](https://kustomize.io/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [SealedSecrets GitHub](https://github.com/bitnami-labs/sealed-secrets)
- [Hologit Documentation](https://github.com/JarvusInnovations/hologit)

### Code for Philly

- **Website**: [codeforphilly.org](https://codeforphilly.org)
- **GitHub**: [github.com/CodeForPhilly](https://github.com/CodeForPhilly)
- **Slack**: [Join Code for Philly Slack](https://codeforphilly.org/chat)

## 🤝 Contributing

Improvements to this documentation are welcome! To contribute:

1. Fork the [cfp-sandbox-cluster repository](https://github.com/CodeForPhilly/cfp-sandbox-cluster)
2. Make changes to files in the `docs/` directory
3. Submit a pull request

Documentation is written in Markdown and published via MkDocs to [codeforphilly.github.io/cfp-sandbox-cluster](https://codeforphilly.github.io/cfp-sandbox-cluster).

## 📅 Next Steps

Ready to deploy? Follow these steps:

1. ✅ Read the [Project Deployment Guide](project-deployment-guide.md)
2. ✅ Set up your project repository with required files
3. ✅ Configure container publishing to ghcr.io
4. ✅ Create Hologit configuration in cluster repo
5. ✅ Set up secrets and deploy
6. ✅ Access your application at `https://your-project.sandbox.k8s.phl.io`

Questions? Ask in #civic-tech-infrastructure on Slack!

---

*Last Updated: November 2025*
