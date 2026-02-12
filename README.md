# GitOps Operators

Cluster operators managed via ArgoCD ApplicationSets with multi-cluster support.

## Structure

```
├── argocd/
│   ├── project.yaml                    # AppProject
│   ├── values.yaml                     # ArgoCD Helm values
│   ├── applicationset.yaml             # Operators (sync-wave: 0)
│   ├── resources-applicationset.yaml   # Post-install CRDs (sync-wave: 1)
│   ├── bootstrap/
│   │   ├── bootstrap.yaml              # startup point
│   │   └── argocd.yaml                 # ArgoCD self-management
│   └── clusters/
│       └── in-cluster.yaml
└── apps/
    |-- service-name/
        |-- config.yaml - (optional) used to point to a helm repo if needed
        |-- clusters/
            |-- {env} - dev is used to test on minikube prd is to deploy in the cluster
                |-- resources - all the manifests to deploy related to the services

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

Override values per cluster environment:

```
apps/metallb/
├── values.yaml                     # Defaults (applies to all environments)
└── clusters/
    ├── dev/
    │   └── values.yaml             # Dev-specific overrides
    └── prd/
        └── values.yaml             # Production-specific overrides
```

## Post-Install Resources

For CRDs and manifests that must deploy after an operator (like MetalLB IPAddressPool):

```
apps/metallb/clusters/prd/resources/
├── kustomization.yaml
├── ip-pool.yaml
└── l2-advertisement.yaml
```

The resources ApplicationSet deploys these with sync-wave "1".

## Environments

- **dev**: Minikube environment (testing)
- **prd**: Production environment (homelab deployment)

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
| tofu-controller | Terraform OpenTofu operator | prd |
| wireguard | VPN server (disabled) | dev |
