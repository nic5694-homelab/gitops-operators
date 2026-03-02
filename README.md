# GitOps Operators

Cluster operators managed via ArgoCD ApplicationSets with multi-cluster support.

## Structure

```
├── argocd/
│   ├── project.yaml                    # AppProject
│   ├── values.yaml                     # ArgoCD Helm values
│   ├── applicationset.yaml             # Operators (sync-wave: 0)
│   ├── resources-applicationset.yaml   # Post-install CRDs (sync-wave: 5)
│   ├── bootstrap/
│   │   ├── base/
│   │   │   ├── kustomization.yaml      # Base configuration
│   │   │   ├── bootstrap.yaml          # GitOps bootstrap app
│   │   │   ├── argocd.yaml             # ArgoCD self-management
│   │   │   └── project.yaml            # operators AppProject
│   │   └── overlays/
│   │       ├── dev/
│   │       │   ├── kustomization.yaml  # Dev patches (targetRevision, cluster secret)
│   │       │   └── cluster-secret.yaml # dev-cluster with environment: dev
│   │       └── prd/
│   │           ├── kustomization.yaml  # Prd patches (targetRevision, cluster secret)
│   │           └── cluster-secret.yaml # prd-cluster with environment: prd
└── apps/
    └── service-name/
        ├── config.yaml                 # Helm chart metadata + environment field
        ├── values.yaml                 # Default values (all environments)
        └── clusters/
            ├── dev/
            │   ├── values.yaml         # Dev-specific overrides
            │   └── resources/          # Dev-specific manifests
            └── prd/
                ├── values.yaml         # Prd-specific overrides
                └── resources/          # Prd-specific manifests
```

## Quick Start

1. Install ArgoCD (first time only):
    ```bash
    helm repo add argo https://argoproj.github.io/argo-helm
    helm repo update

    helm upgrade --install argocd argo/argo-cd \
      --namespace argocd \
      --create-namespace \
      --version 9.3.7
    ```

2. Bootstrap the stack using Kustomize overlays:

   **For dev environment:**
   ```bash
   kubectl kustomize argocd/bootstrap/overlays/dev | kubectl apply -f -
   ```

   **For prd environment:**
   ```bash
   kubectl kustomize argocd/bootstrap/overlays/prd | kubectl apply -f -
   ```

3. Monitor bootstrap progress:
   ```bash
   kubectl -n argocd logs -f deployment/argocd-application-controller
   ```

## Customizing Bootstrap Per Environment

To change the git revision (branch/tag) for an environment, edit `argocd/bootstrap/overlays/{env}/kustomization.yaml`:

```yaml
patches:
  - target:
      kind: Application
      name: bootstrap
    patch: |-
      - op: replace
        path: /spec/source/targetRevision
        value: your-branch-or-tag  # Change this
```

Then redeploy the overlay.

## Adding a New Operator

1. Create `apps/<operator-name>/config.yaml`:
   ```yaml
   name: my-operator
   chart: my-operator
   repoURL: https://charts.example.com
   version: 1.0.0
   namespace: my-operator
   environment: all          # or: dev, prd (controls which environments deploy this app)
   ```

2. Create `apps/<operator-name>/values.yaml`:
   ```yaml
   replicaCount: 2
   resources:
     requests:
       cpu: 100m
       memory: 128Mi
   ```

3. Optionally, create cluster-specific overrides:
   ```bash
   apps/<operator-name>/clusters/{dev,prd}/values.yaml
   ```

4. Commit and push — ApplicationSet auto-generates Applications for all matching environments.

## Environment Field

The `environment` field in `config.yaml` controls deployment scope:

- **`environment: all`** — Deploy to all clusters (default)
- **`environment: dev`** — Deploy only to dev cluster
- **`environment: prd`** — Deploy only to prd cluster

## Environments

- **dev**: Test environment (minikube)
- **prd**: Production environment (homelab server)

Each environment is bootstrapped independently via its own Kustomize overlay (`overlays/dev` or `overlays/prd`), allowing:
- Different git revisions/branches per environment
- Separate cluster secrets with environment labels
- Selective app deployment based on the `environment` field in `config.yaml`

## Operators

| Operator | Description | Environments |
|----------|-------------|--------------|
| adguard | DNS ad blocker & local DNS server | prd |
| cert-manager | Certificate management & Let's Encrypt | dev, prd |
| external-secrets | External secrets synchronization | dev |
| ids | Identity server | prd |
| infisical | Secrets management (disabled) | - |
| ingress-nginx | HTTP/HTTPS ingress controller | dev, prd |
| metallb | Bare metal LoadBalancer | dev, prd |
| minio | S3-compatible object storage | prd |
| wireguard | VPN server (disabled) | dev |
