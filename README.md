# Retail Store GitOps & AKS Managed Karpenter (`retail-gitops`)

This repository is the declarative source of truth for dynamic compute provisioning on Azure Kubernetes Service (AKS). Hosted in **Azure DevOps Repos** and reconciled by **Argo CD** via an App-of-Apps pattern, it bridges the infrastructure boundary from Terraform by configuring **AKS Managed Karpenter (Node Auto-Provisioning)** to scale worker nodes on demand and consolidate them back to zero when idle.

---

## Architecture Overview

Terraform (`02-aks`) provisions only the AKS control plane, OIDC/Workload Identity bindings, and a minimal `system` node pool (`Standard_B2s_v2`) tainted for critical system add-ons. From that point forward, `bootstrap-pipeline.yml` bootstraps Argo CD onto the private cluster using `bootstrap/argocd-values.yaml`, and Argo CD reconciles the Karpenter `AKSNodeClass` and `NodePool` custom resources directly from this repository.

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                        AKS Cluster (retail-store-aks)                   │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │ System Node Pool (Terraform-Managed: Standard_B2s_v2)             │  │
│  │ ├── CoreDNS, Metrics Server, Cilium / Azure CNI                   │  │
│  │ └── Argo CD Control Plane (configured via bootstrap/argocd-values)│  │
│  └─────────────────────────────────┬─────────────────────────────────┘  │
│                                    │ Reconciles bootstrap/root-app.yaml │
│                                    ▼                                    │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │ App-of-Apps: argocd-apps/karpenter-app.yaml (Sync Wave -2)        │  │
│  │ ├── infrastructure/karpenter/aks-nodeclass.yaml                   │  │
│  │ ├── infrastructure/karpenter/nodepool-stateless.yaml              │  │
│  │ └── infrastructure/karpenter/nodepool-stateful.yaml               │  │
│  └─────────────────────────────────┬─────────────────────────────────┘  │
│                                    │ Provisions Nodes on Pending Pods   │
│                     ┌──────────────┴──────────────┐                     │
│                     ▼                             ▼                     │
│  ┌─────────────────────────────────┐ ┌───────────────────────────────┐  │
│  │ Dynamic Worker: stateless-pool  │ │ Dynamic Worker: stateful-pool │  │
│  │ ├── Primary: Standard_E2bs_v5   │ │ ├── Primary: Standard_E2bs_v5 │  │
│  │ ├── Fallbacks: E-series (2vCPU) │ │ ├── Fallbacks: E-series ('s') │  │
│  │ └── Taints: None                │ │ └── Taint: workload=stateful  │  │
│  └─────────────────────────────────┘ └───────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

```text
retail-gitops/
├── argocd-apps/
│   └── karpenter-app.yaml              # Argo CD Application for Karpenter resources (Wave -2)
├── bootstrap/
│   ├── argocd-values.yaml              # Custom Helm values for Argo CD on the system node pool
│   └── root-app.yaml                   # Root Argo CD App-of-Apps manifest pointing to argocd-apps/
├── infrastructure/
│   └── karpenter/                      # AKS Managed Karpenter (NAP) custom resources
│       ├── aks-nodeclass.yaml          # Node image family, OS disk, and Azure VM settings
│       ├── nodepool-stateful.yaml      # Tainted, Premium-SSD worker NodePool
│       └── nodepool-stateless.yaml     # General-purpose stateless worker NodePool
├── bootstrap-pipeline.yml              # Azure DevOps pipeline for automated cluster bootstrap
└── README.md                           # Repository documentation & operational runbooks
```

---

## AKS Managed Karpenter Design (`italynorth`)

AKS Managed Karpenter eliminates static worker VM Scale Sets and Terraform state drift by watching for unschedulable (`Pending`) pods and provisioning right-sized Azure VMs within seconds.

### 1. Two-Pool Workload Isolation (`stateless-pool` vs. `stateful-pool`)

| Pool Name | Intended Workload Profile | Taints | Allowed SKUs (`amd64` / `linux`) | Storage Capability |
| :--- | :--- | :--- | :--- | :--- |
| **`stateless-pool`** | General-purpose, stateless workloads | *None* | `Standard_E2bs_v5`, `Standard_E2bs_v6`, `Standard_E2bds_v5`, `Standard_E2bds_v6`, `Standard_E2_v4` | Standard & Premium Managed Disks |
| **`stateful-pool`** | Stateful workloads requiring persistent volumes | `workload=stateful:NoSchedule` | `Standard_E2bs_v5`, `Standard_E2bs_v6`, `Standard_E2bds_v5`, `Standard_E2bds_v6` | **Strictly `s`-series** (Premium SSD / Azure Managed Disks) |

* **Preventing Cross-Scheduling:** Stateful workloads must specify `nodeSelector: karpenter.sh/nodepool: stateful-pool` alongside a matching `tolerations` block for `workload=stateful:NoSchedule`, ensuring general-purpose pods never schedule onto dedicated stateful nodes.
* **Sync Ordering (`Wave -2`):** `argocd-apps/karpenter-app.yaml` carries `argocd.argoproj.io/sync-wave: "-2"` so `aks-nodeclass.yaml` and both `NodePool`s are registered first, ready to serve future workload waves without scheduling race conditions.

### 2. Multi-Family Quota Resilience

To avoid single-family Azure subscription quota bottlenecks in `italynorth`, both `NodePool` manifests define an ordered fallback list across independent 10-vCPU quota pools (`Standard EBSv5`, `Standard Ebsv6`, `Standard EBDSv5`, and `Standard Ebdsv6`). Karpenter automatically selects the lowest-cost SKU (`Standard_E2bs_v5`, 2 vCPUs / 16 GiB RAM) first and fails over seamlessly if zonal capacity or a family's 10-vCPU cap is reached.

### 3. Scale-to-Zero Cost Optimization

Both `NodePool`s use Karpenter disruption policies (`consolidateAfter: 30s`) to pack underutilized workloads and terminate empty worker VMs within 30 seconds once pods are removed—reducing worker compute cost to zero when idle.

---

## Cluster Bootstrap (Azure DevOps & Private AKS Access)

Because `retail-store-aks` restricts direct public API server access, all cluster bootstrap operations execute inside the private cluster network using `az aks command invoke`.

### Option A: Automated Bootstrap Pipeline (`bootstrap-pipeline.yml`)

Run the **`bootstrap-pipeline.yml`** pipeline in Azure DevOps (`DevOpsLearning` project) after Terraform finishes deploying Layer 2 (`02-aks`). The pipeline automates:

1. Installing or upgrading Argo CD via Helm using `bootstrap/argocd-values.yaml` (scheduling Argo CD components onto the `system` node pool).
2. Creating the `retail-gitops` repository credentials secret in the `argocd` namespace pointing to Azure DevOps Repos.
3. Applying `bootstrap/root-app.yaml` to trigger continuous reconciliation of `argocd-apps/karpenter-app.yaml` and `infrastructure/karpenter/`.

### Option B: Manual Bootstrap via Azure CLI

If bootstrapping manually from a workstation:

1. **Install Argo CD with `bootstrap/argocd-values.yaml`:**
   ```powershell
   az aks command invoke `
     --resource-group "aks-resource-group" `
     --name "retail-store-aks" `
     --file "bootstrap/argocd-values.yaml" `
     --command "helm repo add argo https://argoproj.github.io/argo-helm && helm repo update && helm upgrade --install argocd argo/argo-cd --namespace argocd --create-namespace -f argocd-values.yaml"
   ```

2. **Register the Azure DevOps `retail-gitops` Repository Secret:**
   ```powershell
   az aks command invoke `
     --resource-group "aks-resource-group" `
     --name "retail-store-aks" `
     --command "kubectl create secret generic retail-gitops-repo -n argocd --from-literal=type=git --from-literal=url=https://dev.azure.com/<YOUR_ORG>/DevOpsLearning/_git/retail-gitops --from-literal=username=git --from-literal=password=<YOUR_AZURE_DEVOPS_PAT> --dry-run=client -o yaml | kubectl label --local -f - argocd.argoproj.io/secret-type=repository -o yaml | kubectl apply -f -"
   ```

3. **Apply `bootstrap/root-app.yaml`:**
   ```powershell
   az aks command invoke `
     --resource-group "aks-resource-group" `
     --name "retail-store-aks" `
     --file "bootstrap/root-app.yaml" `
     --command "kubectl apply -f root-app.yaml"
   ```

---

## Day-2 Operations & Verification

### Inspect Karpenter NodePools, NodeClaims, and Provisioned VMs

```powershell
az aks command invoke `
  --resource-group "aks-resource-group" `
  --name "retail-store-aks" `
  --command "kubectl get nodepools; echo '---'; kubectl get nodeclaims -o wide; echo '---'; kubectl get nodes -o wide"
```

### Force an Immediate Argo CD Sync Refresh

```powershell
az aks command invoke `
  --resource-group "aks-resource-group" `
  --name "retail-store-aks" `
  --command "kubectl annotate application karpenter-pools -n argocd argocd.argoproj.io/refresh=hard --overwrite"
```

### Clean Teardown Before `terraform destroy`

Before destroying Layer 2 (`02-aks`) in Terraform, delete all active `NodeClaim` resources so Karpenter cleanly terminates worker VMs and releases subnet NICs in `snet-aks-nodes`:

```powershell
az aks command invoke `
  --resource-group "aks-resource-group" `
  --name "retail-store-aks" `
  --command "kubectl delete nodeclaims --all"
```