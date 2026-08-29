# GitOps Infrastructure & Operators Repository

This repository manages cluster-wide Kubernetes applications, system operators, and core infrastructure using a declarative GitOps workflow using **ArgoCD**, **Kustomize**, and **Helm**.

## Architectural Design

The directory structure and composition model are inspired by the following article: [Red Hat guide on GitOps directory structures](https://developers.redhat.com/articles/2022/09/07/how-set-your-gitops-directory-structure#structuring_your_git_repositories). 

Core principles implemented in this repository:

* **Separation of Concerns:** Environment-agnostic base declarations are strictly separated from environment-specific configurations (overlays).
* **DRY Configuration:** Avoids duplication of raw manifests by utilizing Kustomize's hierarchical composition and inheritance model.
* **Native Tooling Synergy:** Combines Helm's application packaging power with Kustomize's patching framework for declarative, transparent state generation.

---

## Repository Layout

```text
.
├── apps/
│   └── <application-name>/
│       ├── base/
│       │   ├── kustomization.yaml   # Base Helm chart declarations or manifests
│       │   └── values.yaml          # Default upstream configuration values
│       └── overlays/
│           ├── dev/
│           │   └── kustomization.yaml
│           └── prod/
│               ├── kustomization.yaml # Inherits base and applies patches/values
│               ├── svc-patch.yaml     # Targeted resource patches 
│               └── values.yaml        # Environment-specific overrides

```

---

## Adding a New Operator or Application

To introduce a new workload to the cluster following this pattern:

### 1. Create the Directory Structure

Create the isolated base and target overlay directories under `apps/`:

```bash
mkdir -p apps/<app-name>/base apps/<app-name>/overlays/prod

```

### 2. Define the Base Configuration

Create `apps/<app-name>/base/kustomization.yaml` to point to your Helm chart or resource manifests, accompanied by your default `values.yaml`.

### 3. Setup the Production Overlay

Create `apps/<app-name>/overlays/prod/kustomization.yaml` to inherit from the base layer:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

```

### 4. Apply Overlays and Patches

Add environment-specific value files or patch files (such as static IP allocations or node selectors) and register them under `patches:` or `valuesFile:` in your overlay configuration.

### 5. Validate Locally

Test the compilation pipeline using Kustomize with built-in Helm support before committing:

```bash
kustomize build apps/<app-name>/overlays/prod --enable-helm

```

### 6. GitOps Synchronization

Commit and push your changes to the repository. ArgoCD will automatically detect the state change and reconcile the cluster.
