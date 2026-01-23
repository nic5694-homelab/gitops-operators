# GitOps Operators

Cluster operators managed via ArgoCD ApplicationSets with multi-cluster support.

## Structure

```
├── argocd/
│   ├── project.yaml                    # AppProject
│   ├── values.yaml                     # ArgoCD Helm values
│   ├── applicationset.yaml             # Operators (sync-wave: 0)
│   ├── resources-applicationset.yaml   # Post-install CRDs (sync-wave: 1)
│   └── bootstrap/
│       ├── bootstrap.yaml              # Entry point
│       └── argocd.yaml                 # ArgoCD self-management
└── apps/
    ├── cert-manager/
    │   ├── config.yaml
    │   └── values.yaml
    ├── external-secrets/
    │   ├── config.yaml
    │   └── values.yaml
    ├── ingress-nginx/
    │   ├── config.yaml
    │   └── values.yaml
    └── metallb/
        ├── config.yaml
        ├── values.yaml
        └── clusters/
            └── homelab/
                └── resources/
                    ├── kustomization.yaml
                    ├── ip-pool.yaml
                    └── l2-advertisement.yaml
```

## Quick Start

1. Install ArgoCD (first time only):
    ```bash
    helm repo add argo https://argoproj.github.io/argo-helm
    helm repo update

    helm upgrade --install argocd argo/argo-cd \
      --namespace argocd \
      --create-namespace \
      --version 9.2.4
    ```

2. Bootstrap the stack:
   ```bash
   kubectl apply -f argocd/project.yaml
   kubectl apply -f argocd/bootstrap/argocd.yaml
   kubectl apply -f argocd/bootstrap/bootstrap.yaml
   ```

## Adding a New Operator

1. Create `apps/<operator-name>/config.yaml`:
   ```yaml
   name: my-operator
   chart: my-operator
   repoURL: https://charts.example.com
   version: 1.0.0
   namespace: my-operator
   ```

2. Create `apps/<operator-name>/values.yaml`:
   ```yaml
   replicaCount: 2
   resources:
     requests:
       cpu: 100m
       memory: 128Mi
   ```

3. Commit and push — ApplicationSet auto-generates the Application.

## Cluster-Specific Configuration

Override values per cluster:

```
apps/metallb/
├── values.yaml                     # Defaults
└── clusters/
    └── homelab/
        └── values.yaml             # Homelab overrides
```

## Post-Install Resources

For CRDs that must deploy after an operator (like MetalLB IPAddressPool):

```
apps/metallb/clusters/homelab/resources/
├── kustomization.yaml
├── ip-pool.yaml
└── l2-advertisement.yaml
```

The resources ApplicationSet deploys these with sync-wave "1".

## Multi-Cluster

1. Register cluster in ArgoCD
2. Create `apps/<operator>/clusters/<cluster-name>/values.yaml`
3. ApplicationSet auto-generates Applications for each cluster

## Operators

| Operator | Description |
|----------|-------------|
| cert-manager | Certificate management |
| external-secrets | External secrets management |
| ingress-nginx | Ingress controller |
| metallb | Bare metal load balancer |
