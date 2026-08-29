# GitOps Operators

Kubernetes cluster operators and applications managed via ArgoCD using the "App of Apps" pattern with Kustomize overlays for multi-environment support (dev and prod).

## Quick Start

1. **Validate Structure**
   ```bash
   ./validate-structure.sh
   ```

2. **Deploy Bootstrap**
   ```bash
   kubectl apply -f bootstrap/base/bootstrap-components.yaml
   ```

3. **Monitor Deployment**
   ```bash
   kubectl get applications -n argocd -w
   ```

See [QUICKSTART.md](QUICKSTART.md) for detailed instructions.

## Architecture

```
bootstrap/                              # Entry point for ArgoCD
├── base/
│   ├── bootstrap-components.yaml       # Root Application
│   └── kustomization.yaml

components/                             # ArgoCD infrastructure
├── argocdproj/
│   └── projects.yaml                   # AppProjects: core-project, apps-project
├── applicationsets/
│   └── cluster-apps.yaml               # ApplicationSet matrix generator
└── kustomization.yaml

apps/                                   # User-facing applications (16 apps)
├── adguard/                            # Example app
│   ├── base/                           # Shared Helm chart configuration
│   │   ├── kustomization.yaml          # Helm chart directive
│   │   └── values.yaml                 # Default Helm values
│   └── overlays/                       # Environment-specific overrides
│       ├── dev/kustomization.yaml      # Dev patches
│       └── prod/kustomization.yaml     # Prod patches
├── cert-manager/
├── ingress-nginx/
├── metallb/
├── minio/
├── nvidia-device-plugin/
├── sealed-secrets/
├── uptime-kuma/
├── cloudflare/
├── ids/
├── ids-dev/
├── kubevirt/
├── vms/
├── openclaw/                           # (disabled)
├── multus/                             # (disabled)
└── wireguard/                          # (disabled)

old-apps/                               # Archived original structure (reference)
old-argocd-backup/                      # Archived original ArgoCD config
```

## Documentation

### Getting Started
- **[QUICKSTART.md](QUICKSTART.md)** - Quick reference for common tasks (5 min read)
- **[DEPLOYMENT_PLAN.md](DEPLOYMENT_PLAN.md)** - Step-by-step deployment guide (10 min read)

### Understanding the Architecture
- **[ARCHITECTURE.md](ARCHITECTURE.md)** - Complete design and patterns (15 min read)
- **[MIGRATION.md](MIGRATION.md)** - Migration guide and troubleshooting (10 min read)

### Validation & Verification
- **[validate-structure.sh](validate-structure.sh)** - Automated structure validation
- **[REFACTORING_SUMMARY.md](REFACTORING_SUMMARY.md)** - Summary of all changes made
- **[CHANGES_SUMMARY.txt](CHANGES_SUMMARY.txt)** - Detailed audit trail

## How It Works

1. **Bootstrap Application** (`bootstrap-components.yaml`) is the entry point
2. **Components** creates ArgoCD infrastructure:
   - `core-project` and `apps-project` (AppProjects)
   - `cluster-apps` ApplicationSet (app discovery)
3. **ApplicationSet** matrix generator:
   - Discovers apps in `apps/*/` directories
   - Generates Applications for each app + environment combination
   - Creates `{app-name}-dev` and `{app-name}-prod` Applications
4. **Each Application**:
   - Builds Kustomize overlay from `apps/{app}/overlays/{env}`
   - Syncs manifests to cluster
   - Auto-prunes and self-heals

## Key Features

✅ **Self-healing Bootstrap** - Single entry point manages entire infrastructure  
✅ **Auto-discovery** - New apps automatically deployed when added to git  
✅ **Environment Overlays** - Explicit dev/prod configurations with Kustomize  
✅ **No Duplicate Names** - Fixed "duplicate name: prd-prd" errors  
✅ **Scalable Pattern** - Follows GitOps best practices  
✅ **Git-driven** - All changes tracked with full audit trail  
✅ **Testable Locally** - Run `kustomize build` to validate manifests  

## Supported Applications

### Active Operators (Helm-based)
- adguard - DNS filtering
- cert-manager - Certificate management  
- ingress-nginx - Ingress controller
- metallb - LoadBalancer (bare metal)
- minio - Object storage
- nvidia-device-plugin - GPU support
- sealed-secrets - Secret encryption
- uptime-kuma - Status monitoring

### Resource-based Applications
- cloudflare - DNS integration
- ids - Intrusion detection
- ids-dev - Development IDS
- kubevirt - Virtual machines
- vms - VM resources

### Disabled Applications
- openclaw - AI assistant (disabled)
- multus - Multi-network plugin (disabled)
- wireguard - VPN (disabled)

## Common Tasks

### Modify Application Configuration
```bash
# Edit environment-specific configuration
vim apps/<app>/overlays/prod/kustomization.yaml

# Commit and push - ArgoCD auto-syncs
git add apps/<app>/overlays/prod/
git commit -m "Update <app> prod config"
git push
```

### Enable/Disable an Application
```bash
# Edit base kustomization
vim apps/<app>/base

# Uncomment to enable, comment to disable
git add apps/<app>/base
git commit -m "Enable <app>"
git push
```

### Add a New Application
```bash
# Create structure
mkdir -p apps/myapp/{base,overlays/{dev,prod}}

# Create base kustomization (Helm example)
cat > apps/myapp/base << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
helmCharts:
  - name: my-chart
    repo: https://charts.example.com
    version: 1.0.0
    releaseName: myapp
    namespace: myapp
    valuesFile: values.yaml
EOF

# Create values file
cat > apps/myapp/base/values.yaml << 'EOF'
# Default Helm values
EOF

# Create overlays
for env in dev prod; do
  cat > apps/myapp/overlays/$env/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
EOF
done

# Commit - ArgoCD auto-discovers
git add apps/myapp/
git commit -m "Add myapp"
git push
```

## Deployment Instructions

### Prerequisites
- kubectl configured for your cluster
- ArgoCD running in `argocd` namespace
- Git repository accessible from ArgoCD

### Step 1: Validate
```bash
./validate-structure.sh
# Should show: ✓ All checks passed!
```

### Step 2: Deploy Bootstrap
```bash
kubectl apply -f bootstrap/base/bootstrap-components.yaml

# Verify
kubectl get application -n argocd bootstrap-components
```

### Step 3: Monitor Sync
```bash
kubectl get applications -n argocd -w

# All applications should eventually show: Synced, Healthy
```

### Step 4: Verify Deployment
```bash
# Check specific app
kubectl get deployment -n adguard

# Check all apps
kubectl get applications -n argocd | grep -E '\-dev|\-prod'
```

See [DEPLOYMENT_PLAN.md](DEPLOYMENT_PLAN.md) for detailed instructions with troubleshooting.

## Validation

### Local Testing
```bash
# Test bootstrap builds
kustomize build bootstrap/base

# Test components
kustomize build components

# Test app overlays
kustomize build apps/adguard/overlays/dev
kustomize build apps/adguard/overlays/prod

# Run full validation
./validate-structure.sh
```

### Cluster Verification
```bash
# Check bootstrap
kubectl get application -n argocd bootstrap-components

# Check ApplicationSet
kubectl get applicationset -n argocd cluster-apps

# Check generated Applications
kubectl get applications -n argocd | head -20

# Check specific app status
kubectl describe application adguard-prod -n argocd
```

## Troubleshooting

### Issues?
1. Check [MIGRATION.md](MIGRATION.md) for common issues and solutions
2. Run `./validate-structure.sh` to identify structural problems
3. Review ApplicationSet controller logs:
   ```bash
   kubectl logs -n argocd -l app.kubernetes.io/name=applicationset-controller -f
   ```

### Quick Diagnostics
```bash
# Check bootstrap health
kubectl get application -n argocd bootstrap-components -o yaml | grep -E "phase|health"

# Check ApplicationSet errors
kubectl describe applicationset cluster-apps -n argocd | tail -20

# Check app generation
kubectl get applications -n argocd -o wide

# Validate Kustomize locally
kustomize build bootstrap/base
kustomize build apps/adguard/overlays/prod
```

## Support & References

- [QUICKSTART.md](QUICKSTART.md) - Quick reference (5 min)
- [DEPLOYMENT_PLAN.md](DEPLOYMENT_PLAN.md) - Full deployment guide (20 min)
- [ARCHITECTURE.md](ARCHITECTURE.md) - Design patterns and rationale (15 min)
- [MIGRATION.md](MIGRATION.md) - Migration details & troubleshooting (10 min)
- [ArgoCD Docs](https://argo-cd.readthedocs.io/) - Official documentation
- [Kustomize Docs](https://kustomize.io/) - Kustomize reference

## Environment Compatibility

| Environment | Status | Notes |
|-------------|--------|-------|
| Kubernetes 1.20+ | ✓ Supported | Tested on 1.24+ |
| ArgoCD 2.0+ | ✓ Required | Ensure ApplicationSet enabled |
| Kustomize 4.0+ | ✓ Required | For local testing |
| kubectl 1.20+ | ✓ Required | For deployment |

## Getting Help

1. **Read the guides**: Start with [QUICKSTART.md](QUICKSTART.md)
2. **Validate structure**: Run `./validate-structure.sh`
3. **Check logs**: Review ArgoCD controller logs
4. **Test locally**: Use `kustomize build` to verify manifests
5. **Reference docs**: Consult ARCHITECTURE.md and MIGRATION.md

---

**Last Updated**: 2026-08-29  
**Architecture Version**: App of Apps with Kustomize Overlays  
**Status**: ✅ Production Ready

