# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Repository Overview

This is a GitOps-managed Kubernetes homelab repository using Flux CD for continuous deployment. The repository manages applications and monitoring infrastructure for a staging Kubernetes cluster using the GitOps pattern where git commits automatically trigger deployments.

## Architecture

### GitOps Flow
- **Git Repository**: `git@github.com:iampavelk/k8s-homelab.git`
- **Flux CD**: Automatically syncs changes from the `main` branch every 1 minute
- **Cluster Path**: `./clusters/staging` is the root path monitored by Flux
- **Secret Management**: SOPS (Secrets OPerationS) with Age encryption for managing sensitive data

### Directory Structure
```
├── apps/                       # Application definitions
│   ├── base/                  # Base Kustomize configurations
│   │   └── linkding/         # Linkding bookmark manager app
│   └── staging/              # Environment-specific overlays
│       └── linkding/         # Staging-specific config and secrets
├── clusters/                  # Cluster configurations
│   └── staging/              # Staging cluster definitions
│       ├── flux-system/      # Flux CD system components
│       ├── apps.yaml         # Apps Kustomization
│       └── monitoring.yaml   # Monitoring Kustomization
└── monitoring/               # Monitoring stack
    └── controllers/          # Monitoring controllers
        ├── base/            # Base Helm releases
        └── staging/         # Staging environment config
```

## Common Commands

### Flux CD Operations
```bash
# Check Flux system status
kubectl get kustomizations -A

# Check Flux source status
kubectl get gitrepository -A

# Force reconciliation of Flux sources
kubectl annotate gitrepository flux-system -n flux-system reconcile.fluxcd.io/requestedAt="$(date +%s)"

# Check Flux logs
kubectl logs -n flux-system -l app=source-controller -f
kubectl logs -n flux-system -l app=kustomize-controller -f
kubectl logs -n flux-system -l app=helm-controller -f
```

### Secret Management (SOPS)
```bash
# Encrypt a new secret file
sops -e -i apps/staging/linkding/new-secret.yaml

# Edit an encrypted secret
sops apps/staging/linkding/linkding-container-env-secret.yaml

# Decrypt and view secret (for debugging)
sops -d apps/staging/linkding/linkding-container-env-secret.yaml
```

### Kustomize Testing
```bash
# Test kustomize build before committing
kustomize build apps/staging/linkding/
kustomize build monitoring/controllers/staging/
kustomize build clusters/staging/

# Validate YAML syntax
kubectl apply --dry-run=client -k apps/staging/linkding/
```

### Application Management
```bash
# Check application status
kubectl get pods -n linkding
kubectl get ingress -n linkding

# View application logs
kubectl logs -n linkding -l app=linkding -f

# Check monitoring stack
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

## Key Configuration Details

### SOPS Configuration
- Age public key: `age1qm35vx6cq8nl74rcnltctarkng9qra9ch2mhsxzsqrf38z6efvdsdzhku2`
- Encrypts `data` and `stringData` fields in YAML files
- Configuration file: `clusters/staging/.sops.yaml`

### Applications
- **Linkding**: Self-hosted bookmark manager running on port 9090
  - Domain: `linkding.kprcloud.org`
  - Ingress: Traefik
  - Storage: 1Gi PVC
  - Image: `sissbruecker/linkding:1.41.0`

### Monitoring
- **Kube-Prometheus-Stack**: Complete monitoring solution using Helm
  - Version: `76.0.*`
  - Grafana admin password: `q1w2e3R$` (should be changed in production)
  - Namespace: `monitoring`

## Development Workflow

1. **Making Changes**: Edit YAML files in the appropriate environment directory
2. **Secrets**: Use `sops -e -i filename.yaml` to encrypt any secrets before committing
3. **Testing**: Run `kustomize build` to validate configurations locally
4. **Deployment**: Commit and push changes to `main` branch
5. **Verification**: Flux will automatically reconcile within 1-10 minutes

## Important Notes

- All secret files must be encrypted with SOPS before committing
- The Flux system automatically prunes resources not defined in git
- Changes to the `main` branch are automatically deployed to the staging cluster
- Kustomize overlays follow the base/environment pattern
- Ingress uses Traefik ingress controller
- Domain: `*.kprcloud.org` is used for external access

## Troubleshooting

### Common Issues
- **Secrets not decrypting**: Ensure SOPS Age key is properly configured in the cluster
- **Flux not syncing**: Check GitRepository and Kustomization resources for errors
- **Pod startup failures**: Often related to secret mounting or PVC binding issues
- **Ingress not working**: Verify Traefik controller and DNS configuration

### Debug Commands
```bash
# Check Flux reconciliation status
kubectl get kustomizations -A -o wide

# View detailed Flux events
kubectl describe kustomization apps -n flux-system

# Check if secrets are properly mounted
kubectl describe pod -n linkding -l app=linkding
```
