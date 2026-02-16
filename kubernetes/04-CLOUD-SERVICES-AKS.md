# Azure AKS — Azure Kubernetes Service

## Table of Contents

1. [Overview](#overview)
2. [AKS Architecture](#aks-architecture)
   - [Service Tiers](#service-tiers)
   - [Control Plane](#control-plane)
   - [Node Pools](#node-pools-architecture)
3. [Prerequisites and Azure Setup](#prerequisites-and-azure-setup)
   - [Required Tools](#required-tools)
   - [Azure Subscription and Resource Groups](#azure-subscription-and-resource-groups)
4. [Creating an AKS Cluster](#creating-an-aks-cluster)
   - [Using Azure CLI](#using-azure-cli)
   - [Using Azure Portal](#using-azure-portal)
   - [Using Terraform](#using-terraform)
   - [Using Bicep](#using-bicep)
5. [Networking](#networking)
   - [kubenet vs Azure CNI](#kubenet-vs-azure-cni)
   - [Azure CNI Overlay](#azure-cni-overlay)
   - [Bring-Your-Own CNI](#bring-your-own-cni)
   - [Virtual Network Design](#virtual-network-design)
6. [Node Pools](#node-pools)
   - [System vs User Node Pools](#system-vs-user-node-pools)
   - [Spot Node Pools](#spot-node-pools)
   - [Virtual Nodes (ACI)](#virtual-nodes-aci)
7. [Storage](#storage)
   - [Azure Disk CSI Driver](#azure-disk-csi-driver)
   - [Azure File CSI Driver](#azure-file-csi-driver)
   - [Azure Blob CSI Driver](#azure-blob-csi-driver)
   - [StorageClasses](#storageclasses)
8. [Load Balancing](#load-balancing)
   - [Azure Load Balancer](#azure-load-balancer)
   - [Application Gateway Ingress Controller (AGIC)](#application-gateway-ingress-controller-agic)
9. [Authentication and Authorization](#authentication-and-authorization)
   - [Microsoft Entra ID Integration](#microsoft-entra-id-integration)
   - [Azure RBAC for Kubernetes](#azure-rbac-for-kubernetes)
   - [Managed Identities](#managed-identities)
   - [Workload Identity](#workload-identity)
10. [Cluster Autoscaler and KEDA](#cluster-autoscaler-and-keda)
    - [Cluster Autoscaler](#cluster-autoscaler)
    - [KEDA](#keda)
11. [Monitoring](#monitoring)
    - [Azure Monitor Container Insights](#azure-monitor-container-insights)
    - [Managed Prometheus and Grafana](#managed-prometheus-and-grafana)
12. [AKS Add-ons and Extensions](#aks-add-ons-and-extensions)
13. [Cost Management](#cost-management)
14. [Best Practices](#best-practices)
15. [Next Steps](#next-steps)

## Overview

This document covers **Azure Kubernetes Service (AKS)** — Microsoft Azure's managed Kubernetes offering that provisions and manages the control plane, automates Kubernetes version upgrades and patching, and integrates natively with Azure networking, identity, storage, and observability services.

### Target Audience

- DevOps and platform engineers deploying workloads on Microsoft Azure
- Developers building and shipping containerized applications to AKS
- SREs operating production Kubernetes clusters on Azure
- Architects evaluating managed Kubernetes options across cloud providers

### Scope

- AKS architecture, service tiers, control plane, and node pool options
- Cluster creation with Azure CLI, Azure Portal, Terraform, and Bicep
- Networking (Azure CNI, kubenet, CNI Overlay), storage (Disk/File/Blob CSI), and load balancing
- Microsoft Entra ID integration, Azure RBAC, managed identities, and workload identity
- Autoscaling with Cluster Autoscaler and KEDA
- Monitoring, add-ons, cost management, and operational best practices

## AKS Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Azure Region                                 │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │            AKS Control Plane (Azure-managed)                │   │
│  │                                                              │   │
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐                │   │
│  │  │ API      │   │ API      │   │ API      │   HA across    │   │
│  │  │ Server   │   │ Server   │   │ Server   │   3 AZs        │   │
│  │  └────┬─────┘   └────┬─────┘   └────┬─────┘                │   │
│  │       │              │              │                        │   │
│  │  ┌────▼──────────────▼──────────────▼────┐                  │   │
│  │  │           etcd (managed)               │                  │   │
│  │  └───────────────────────────────────────┘                  │   │
│  └───────────────────────┬──────────────────────────────────────┘   │
│                          │                                          │
│  ┌───────────────────────▼──────────────────────────────────────┐   │
│  │              Data Plane (customer-managed)                   │   │
│  │                                                              │   │
│  │  AZ-1               AZ-2               AZ-3                 │   │
│  │  ┌──────────┐      ┌──────────┐      ┌──────────┐          │   │
│  │  │  Node    │      │  Node    │      │ Virtual  │          │   │
│  │  │ ┌──┐┌──┐│      │ ┌──┐┌──┐│      │  Node    │          │   │
│  │  │ │P1││P2││      │ │P3││P4││      │  (ACI)   │          │   │
│  │  │ └──┘└──┘│      │ └──┘└──┘│      │  ┌──┐    │          │   │
│  │  └──────────┘      └──────────┘      │  │P5│    │          │   │
│  │                                       │  └──┘    │          │   │
│  │                                       └──────────┘          │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Service Tiers

AKS offers three service tiers that determine control plane capabilities and SLA guarantees:

| Tier | SLA | Features | Cost |
|---|---|---|---|
| **Free** | None (best-effort) | Single control plane instance, basic management | Free |
| **Standard** | 99.95 % (AZ) / 99.9 % (non-AZ) | HA control plane, Cluster Autoscaler, uptime SLA | Per cluster/hour |
| **Premium** | 99.95 % (AZ) / 99.9 % (non-AZ) | Everything in Standard + long-term support (LTS) versions, Microsoft-managed safeguards | Per cluster/hour |

### Control Plane

Azure fully manages the AKS control plane. You do **not** provision, patch, or scale API servers or etcd. Key characteristics:

| Aspect | Detail |
|---|---|
| **API servers** | Distributed across Availability Zones (Standard/Premium tiers) |
| **etcd** | Managed, encrypted at rest, backed up automatically |
| **Upgrades** | Azure provides in-place Kubernetes version upgrades with node-surge support |
| **Endpoint access** | Public, private, or both (configurable via `--api-server-authorized-ip-ranges` or private cluster) |
| **Cost** | Control plane is **free** — you pay for worker nodes and optional tier upgrades |

### Node Pools (Architecture)

You choose how to run your workloads:

| Option | Description | Use Case |
|---|---|---|
| **System node pool** | Runs system Pods (CoreDNS, metrics-server, etc.) — at least one required | Cluster infrastructure components |
| **User node pool** | Runs application workloads, can use different VM sizes | General-purpose, GPU, memory-optimized workloads |
| **Spot node pool** | Uses Azure Spot VMs at significant discount, can be evicted | Fault-tolerant, batch, dev/test workloads |
| **Virtual nodes (ACI)** | Serverless — each Pod runs on Azure Container Instances | Burst workloads, rapid scale-out |

## Prerequisites and Azure Setup

### Required Tools

Install the following CLI tools before working with AKS:

```bash
# Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Install the aks-preview extension (optional, for preview features)
az extension add --name aks-preview

# kubectl (installed via Azure CLI)
az aks install-cli

# kubelogin (for Entra ID / AAD authentication)
az aks install-cli  # installs both kubectl and kubelogin

# Verify installations
az version
kubectl version --client
kubelogin --version
```

### Azure Subscription and Resource Groups

AKS resources live inside a **resource group**. Create one before provisioning a cluster:

```bash
# Log in to Azure
az login

# Set the active subscription
az account set --subscription "<subscription-name-or-id>"

# Create a resource group
az group create \
  --name rg-aks-production \
  --location eastus

# Register required resource providers (usually registered by default)
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.OperationsManagement
az provider register --namespace Microsoft.OperationalInsights
```

## Creating an AKS Cluster

### Using Azure CLI

**Quick start (defaults):**

```bash
az aks create \
  --resource-group rg-aks-production \
  --name production-cluster \
  --location eastus \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --generate-ssh-keys
```

**Production-ready cluster with advanced options:**

```bash
az aks create \
  --resource-group rg-aks-production \
  --name production-cluster \
  --location eastus \
  --kubernetes-version 1.31 \
  --tier standard \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --zones 1 2 3 \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --network-dataplane cilium \
  --pod-cidr 192.168.0.0/16 \
  --vnet-subnet-id "/subscriptions/<sub>/resourceGroups/rg-aks-production/providers/Microsoft.Network/virtualNetworks/aks-vnet/subnets/aks-subnet" \
  --enable-managed-identity \
  --enable-aad \
  --enable-azure-rbac \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10 \
  --enable-addons monitoring \
  --enable-msi-auth-for-monitoring \
  --generate-ssh-keys \
  --tags Environment=production Team=platform

# Get credentials and verify
az aks get-credentials \
  --resource-group rg-aks-production \
  --name production-cluster

kubectl get nodes
```

### Using Azure Portal

Step-by-step process through the Azure web portal:

1. Navigate to **Kubernetes services** in the Azure portal
2. Click **Create → Kubernetes cluster**
3. **Basics** tab:
   - **Subscription:** select your subscription
   - **Resource group:** `rg-aks-production`
   - **Cluster name:** `production-cluster`
   - **Region:** `East US`
   - **Availability zones:** Zones 1, 2, 3
   - **AKS pricing tier:** Standard
   - **Kubernetes version:** select the latest supported version (e.g., `1.31`)
4. **Node pools** tab:
   - **System node pool:** `Standard_D4s_v5`, node count 3, autoscaling enabled (3–10)
   - Optionally add a **User node pool** for application workloads
5. **Networking** tab:
   - **Network configuration:** Azure CNI Overlay
   - **Network policy:** Cilium
   - **DNS name prefix:** `production-cluster`
6. **Integrations** tab:
   - **Container registry:** attach an existing ACR
   - **Azure Monitor:** enable Container Insights
7. **Advanced** tab:
   - **Infrastructure resource group:** optionally set a custom name
8. Click **Review + create**, then **Create**
9. Once provisioned, click **Connect** and follow the `az aks get-credentials` command

### Using Terraform

Terraform provides a declarative, version-controlled approach to infrastructure. The example below uses the AzureRM provider.

```hcl
# main.tf
terraform {
  required_version = ">= 1.5"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# ---------- Resource Group ----------
resource "azurerm_resource_group" "aks" {
  name     = "rg-aks-production"
  location = var.location
}

# ---------- Virtual Network ----------
resource "azurerm_virtual_network" "aks" {
  name                = "vnet-aks"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.aks.location
  resource_group_name = azurerm_resource_group.aks.name
}

resource "azurerm_subnet" "aks_nodes" {
  name                 = "snet-aks-nodes"
  resource_group_name  = azurerm_resource_group.aks.name
  virtual_network_name = azurerm_virtual_network.aks.name
  address_prefixes     = ["10.0.1.0/24"]
}

# ---------- AKS Cluster ----------
resource "azurerm_kubernetes_cluster" "main" {
  name                = var.cluster_name
  location            = azurerm_resource_group.aks.location
  resource_group_name = azurerm_resource_group.aks.name
  dns_prefix          = var.cluster_name
  kubernetes_version  = "1.31"
  sku_tier            = "Standard"

  default_node_pool {
    name                = "system"
    vm_size             = "Standard_D4s_v5"
    auto_scaling_enabled = true
    min_count           = 3
    max_count           = 10
    zones               = [1, 2, 3]
    vnet_subnet_id      = azurerm_subnet.aks_nodes.id
    os_disk_size_gb     = 128

    node_labels = {
      "nodepool-type" = "system"
    }
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin      = "azure"
    network_plugin_mode = "overlay"
    network_dataplane   = "cilium"
    pod_cidr            = "192.168.0.0/16"
    service_cidr        = "10.1.0.0/16"
    dns_service_ip      = "10.1.0.10"
  }

  azure_active_directory_role_based_access_control {
    azure_rbac_enabled = true
    tenant_id          = var.tenant_id
  }

  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.aks.id
  }

  tags = {
    Environment = "production"
    Team        = "platform"
  }
}

# ---------- User Node Pool ----------
resource "azurerm_kubernetes_cluster_node_pool" "applications" {
  name                  = "apps"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.main.id
  vm_size               = "Standard_D8s_v5"
  auto_scaling_enabled  = true
  min_count             = 1
  max_count             = 20
  zones                 = [1, 2, 3]
  vnet_subnet_id        = azurerm_subnet.aks_nodes.id
  mode                  = "User"

  node_labels = {
    "nodepool-type" = "applications"
  }

  node_taints = []
}

# ---------- Log Analytics ----------
resource "azurerm_log_analytics_workspace" "aks" {
  name                = "law-aks-production"
  location            = azurerm_resource_group.aks.location
  resource_group_name = azurerm_resource_group.aks.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}

# ---------- Variables ----------
variable "location" {
  default = "eastus"
}

variable "cluster_name" {
  default = "production-cluster"
}

variable "tenant_id" {
  description = "Azure AD / Entra ID tenant ID"
}

# ---------- Outputs ----------
output "kube_config" {
  value     = azurerm_kubernetes_cluster.main.kube_config_raw
  sensitive = true
}

output "cluster_fqdn" {
  value = azurerm_kubernetes_cluster.main.fqdn
}
```

```bash
terraform init
terraform plan -var tenant_id="<your-tenant-id>"
terraform apply -var tenant_id="<your-tenant-id>"

# Get credentials
az aks get-credentials \
  --resource-group rg-aks-production \
  --name production-cluster
```

### Using Bicep

Bicep is Azure's domain-specific language for deploying Azure resources declaratively.

```bicep
// aks-cluster.bicep
@description('Location for all resources')
param location string = resourceGroup().location

@description('AKS cluster name')
param clusterName string = 'production-cluster'

@description('Kubernetes version')
param kubernetesVersion string = '1.31'

@description('System node pool VM size')
param nodeVmSize string = 'Standard_D4s_v5'

@description('Log Analytics workspace name')
param logAnalyticsName string = 'law-aks-production'

// Log Analytics Workspace
resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2023-09-01' = {
  name: logAnalyticsName
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 30
  }
}

// AKS Cluster
resource aksCluster 'Microsoft.ContainerService/managedClusters@2024-09-01' = {
  name: clusterName
  location: location
  sku: {
    name: 'Base'
    tier: 'Standard'
  }
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    kubernetesVersion: kubernetesVersion
    dnsPrefix: clusterName
    agentPoolProfiles: [
      {
        name: 'system'
        count: 3
        vmSize: nodeVmSize
        osType: 'Linux'
        mode: 'System'
        enableAutoScaling: true
        minCount: 3
        maxCount: 10
        availabilityZones: [ '1', '2', '3' ]
        osDiskSizeGB: 128
        type: 'VirtualMachineScaleSets'
        nodeLabels: {
          'nodepool-type': 'system'
        }
      }
    ]
    networkProfile: {
      networkPlugin: 'azure'
      networkPluginMode: 'overlay'
      networkDataplane: 'cilium'
      podCidr: '192.168.0.0/16'
      serviceCidr: '10.1.0.0/16'
      dnsServiceIP: '10.1.0.10'
    }
    aadProfile: {
      managed: true
      enableAzureRBAC: true
    }
    addonProfiles: {
      omsagent: {
        enabled: true
        config: {
          logAnalyticsWorkspaceResourceID: logAnalytics.id
        }
      }
    }
  }
}

output clusterName string = aksCluster.name
output controlPlaneFQDN string = aksCluster.properties.fqdn
```

```bash
# Deploy with Azure CLI
az deployment group create \
  --resource-group rg-aks-production \
  --template-file aks-cluster.bicep \
  --parameters clusterName=production-cluster
```

## Networking

### kubenet vs Azure CNI

AKS supports multiple networking models. The two foundational options are **kubenet** and **Azure CNI**:

| Feature | kubenet | Azure CNI |
|---|---|---|
| **Pod IPs** | NAT'd behind node IP (pod CIDR separate) | Pods get IPs directly from the VNet subnet |
| **IP consumption** | Low — only node IPs from the subnet | High — one IP per Pod from the subnet |
| **Performance** | Extra hop (bridge) adds latency | Native VNet performance |
| **Network policies** | Calico only | Azure or Calico |
| **Max pods per node** | 110 (default) | 30 (default, configurable to 250) |
| **Best for** | Small clusters, IP-constrained environments | Production, hybrid networking, direct Pod access |

### Azure CNI Overlay

**Azure CNI Overlay** combines the benefits of Azure CNI performance with efficient IP management. Pods receive IPs from a private CIDR that is **not** routable on the VNet, reducing subnet IP pressure.

```bash
# Create a cluster with CNI Overlay and Cilium dataplane
az aks create \
  --resource-group rg-aks-production \
  --name overlay-cluster \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --network-dataplane cilium \
  --pod-cidr 192.168.0.0/16 \
  --node-count 3 \
  --generate-ssh-keys
```

| Feature | Azure CNI (traditional) | Azure CNI Overlay |
|---|---|---|
| **Pod IP source** | VNet subnet | Private overlay CIDR |
| **Subnet IP usage** | 1 IP per Pod | 1 IP per Node only |
| **Max Pods per node** | 30 (default) | 250 (default) |
| **Pod-to-VNet routing** | Direct | NAT at node boundary |
| **Best for** | Direct Pod reachability | Large clusters, IP conservation |

### Bring-Your-Own CNI

AKS supports deploying with **no CNI pre-installed**, allowing you to install your own (e.g., Cilium, Calico, Antrea):

```bash
# Create a cluster without a CNI plugin
az aks create \
  --resource-group rg-aks-production \
  --name byocni-cluster \
  --network-plugin none \
  --node-count 3 \
  --generate-ssh-keys

# Then install your preferred CNI (example: Cilium via Helm)
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set aksbyocni.enabled=true \
  --set nodeinit.enabled=true
```

### Virtual Network Design

A production-ready AKS VNet typically includes dedicated subnets:

```
┌───────────────────────────────────────────────────────────┐
│  VNet: 10.0.0.0/16                                        │
│                                                           │
│  ┌─────────────────────┐  ┌─────────────────────┐        │
│  │ snet-aks-nodes      │  │ snet-aks-pods        │        │
│  │ 10.0.1.0/24         │  │ 10.0.4.0/22          │        │
│  │ (node IPs)          │  │ (pod IPs — CNI only) │        │
│  └─────────────────────┘  └─────────────────────┘        │
│                                                           │
│  ┌─────────────────────┐  ┌─────────────────────┐        │
│  │ snet-appgw           │  │ snet-privlink        │        │
│  │ 10.0.2.0/24         │  │ 10.0.3.0/24          │        │
│  │ (App Gateway)       │  │ (Private Endpoints)  │        │
│  └─────────────────────┘  └─────────────────────┘        │
└───────────────────────────────────────────────────────────┘
```

Key design recommendations:

- Size node subnets for the maximum number of nodes plus headroom for upgrades (surge nodes)
- Use a `/22` or larger for pod subnets when using traditional Azure CNI
- Place Azure Application Gateway in its own dedicated subnet
- Use Private Endpoints (Private Link) for Azure PaaS services (ACR, Key Vault, SQL)
- Peer VNets or use Azure Virtual WAN for multi-region or hybrid connectivity

## Node Pools

### System vs User Node Pools

Every AKS cluster requires at least one **system node pool**. Additional **user node pools** isolate application workloads.

```bash
# Add a user node pool for application workloads
az aks nodepool add \
  --resource-group rg-aks-production \
  --cluster-name production-cluster \
  --name apps \
  --mode User \
  --node-count 3 \
  --node-vm-size Standard_D8s_v5 \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 20 \
  --labels workload=applications

# Add a GPU node pool
az aks nodepool add \
  --resource-group rg-aks-production \
  --cluster-name production-cluster \
  --name gpu \
  --mode User \
  --node-count 1 \
  --node-vm-size Standard_NC6s_v3 \
  --node-taints "sku=gpu:NoSchedule" \
  --labels workload=gpu
```

| Aspect | System Node Pool | User Node Pool |
|---|---|---|
| **Purpose** | Runs critical system Pods (CoreDNS, metrics-server) | Runs application workloads |
| **Required** | At least one per cluster | Optional |
| **Taint** | `CriticalAddonsOnly=true:NoSchedule` (auto-applied) | User-defined or none |
| **Min nodes** | 1 (2 recommended for production) | 0 (can scale to zero) |
| **Scale to zero** | Not allowed | Allowed |

### Spot Node Pools

Spot node pools use **Azure Spot VMs** at up to 90 % discount. Nodes can be evicted when Azure needs capacity.

```bash
az aks nodepool add \
  --resource-group rg-aks-production \
  --cluster-name production-cluster \
  --name spot \
  --mode User \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 20 \
  --labels "kubernetes.azure.com/scalesetpriority=spot" \
  --node-taints "kubernetes.azure.com/scalesetpriority=spot:NoSchedule"
```

Schedule tolerant workloads on Spot nodes using tolerations:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-processor
spec:
  replicas: 5
  selector:
    matchLabels:
      app: batch-processor
  template:
    metadata:
      labels:
        app: batch-processor
    spec:
      tolerations:
        - key: "kubernetes.azure.com/scalesetpriority"
          operator: "Equal"
          value: "spot"
          effect: "NoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: "kubernetes.azure.com/scalesetpriority"
                    operator: In
                    values:
                      - "spot"
      containers:
        - name: processor
          image: myacr.azurecr.io/batch-processor:latest
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"
```

### Virtual Nodes (ACI)

Virtual nodes provide serverless burst capacity through **Azure Container Instances (ACI)**. Pods start in seconds without pre-provisioning VMs.

```bash
# Enable virtual nodes (requires Azure CNI networking)
az aks enable-addons \
  --resource-group rg-aks-production \
  --name production-cluster \
  --addons virtual-node \
  --subnet-name snet-aci

# Schedule a Pod on virtual nodes
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: burst-workload
spec:
  nodeSelector:
    kubernetes.io/role: agent
    type: virtual-kubelet
  tolerations:
    - key: virtual-kubelet.io/provider
      operator: Exists
    - key: azure.com/aci
      effect: NoSchedule
  containers:
    - name: app
      image: nginx:latest
      resources:
        requests:
          cpu: "1"
          memory: "2Gi"
EOF
```

## Storage

### Azure Disk CSI Driver

Azure Disk CSI provides block storage backed by Azure Managed Disks. Best for stateful workloads requiring high-performance, single-node access.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-csi-premium
  resources:
    requests:
      storage: 100Gi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: managed-csi-premium
        resources:
          requests:
            storage: 100Gi
```

### Azure File CSI Driver

Azure File CSI provides shared file storage (SMB/NFS) backed by Azure Files. Best for workloads that need `ReadWriteMany` access.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-content
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile-csi-premium
  resources:
    requests:
      storage: 50Gi
```

### Azure Blob CSI Driver

Azure Blob CSI mounts Azure Blob Storage containers as a filesystem (via BlobFuse2 or NFS 3.0). Best for large unstructured data.

```bash
# Install the Blob CSI driver add-on
az aks update \
  --resource-group rg-aks-production \
  --name production-cluster \
  --enable-blob-driver
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: blob-data
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azureblob-fuse-premium
  resources:
    requests:
      storage: 200Gi
```

### StorageClasses

AKS ships with several built-in StorageClasses:

| StorageClass | Backing Store | Access Mode | Use Case |
|---|---|---|---|
| `managed-csi` | Standard SSD Managed Disk | ReadWriteOnce | General-purpose block storage |
| `managed-csi-premium` | Premium SSD Managed Disk | ReadWriteOnce | Production databases, high IOPS |
| `azurefile-csi` | Azure Files (Standard) | ReadWriteMany | Shared file access |
| `azurefile-csi-premium` | Azure Files (Premium) | ReadWriteMany | High-perf shared file access |
| `azureblob-fuse-premium` | Azure Blob (BlobFuse2) | ReadWriteMany | Large datasets, ML training data |
| `azureblob-nfs-premium` | Azure Blob (NFS 3.0) | ReadWriteMany | Linux NFS workloads on Blob |

Custom StorageClass example with zone-redundant Premium SSD v2:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-zrs
provisioner: disk.csi.azure.com
parameters:
  skuName: PremiumV2_LRS
  cachingmode: None
  DiskIOPSReadWrite: "5000"
  DiskMBpsReadWrite: "200"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

## Load Balancing

### Azure Load Balancer

AKS uses the **Azure Load Balancer** (Standard SKU) by default for `LoadBalancer`-type Services. External and internal load balancers are supported.

```yaml
# External Load Balancer (default)
apiVersion: v1
kind: Service
metadata:
  name: web-external
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
---
# Internal Load Balancer
apiVersion: v1
kind: Service
metadata:
  name: api-internal
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
    - port: 443
      targetPort: 8443
```

### Application Gateway Ingress Controller (AGIC)

**AGIC** uses Azure Application Gateway as a Layer 7 ingress controller with WAF, SSL termination, and URL-based routing.

```bash
# Enable the AGIC add-on with a new Application Gateway
az aks enable-addons \
  --resource-group rg-aks-production \
  --name production-cluster \
  --addons ingress-appgw \
  --appgw-name aks-appgw \
  --appgw-subnet-cidr "10.0.2.0/24"
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    appgw.ingress.kubernetes.io/backend-protocol: "http"
spec:
  tls:
    - hosts:
        - app.example.com
      secretName: tls-secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-frontend
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-backend
                port:
                  number: 8080
```

## Authentication and Authorization

### Microsoft Entra ID Integration

AKS integrates with **Microsoft Entra ID** (formerly Azure AD) for user authentication. When enabled, `kubectl` authenticates via Entra ID tokens through `kubelogin`.

```bash
# Enable managed Entra ID integration on an existing cluster
az aks update \
  --resource-group rg-aks-production \
  --name production-cluster \
  --enable-aad

# Assign a cluster admin group
az aks update \
  --resource-group rg-aks-production \
  --name production-cluster \
  --aad-admin-group-object-ids "<entra-group-object-id>"

# Authenticate with kubelogin
az aks get-credentials \
  --resource-group rg-aks-production \
  --name production-cluster

kubelogin convert-kubeconfig -l azurecli
kubectl get nodes
```

### Azure RBAC for Kubernetes

With **Azure RBAC for Kubernetes**, you manage Kubernetes authorization directly through Azure role assignments — no separate `ClusterRoleBinding` objects needed.

```bash
# Enable Azure RBAC for Kubernetes authorization
az aks update \
  --resource-group rg-aks-production \
  --name production-cluster \
  --enable-azure-rbac

# Grant a user the built-in "Azure Kubernetes Service Cluster User Role"
az role assignment create \
  --assignee "<user-or-group-object-id>" \
  --role "Azure Kubernetes Service Cluster User Role" \
  --scope "/subscriptions/<sub>/resourceGroups/rg-aks-production/providers/Microsoft.ContainerService/managedClusters/production-cluster"

# Grant namespace-scoped access
az role assignment create \
  --assignee "<user-or-group-object-id>" \
  --role "Azure Kubernetes Service RBAC Writer" \
  --scope "/subscriptions/<sub>/resourceGroups/rg-aks-production/providers/Microsoft.ContainerService/managedClusters/production-cluster/namespaces/app-team"
```

Built-in Azure RBAC roles for Kubernetes:

| Role | Permissions |
|---|---|
| **Azure Kubernetes Service RBAC Reader** | Read-only access to most objects in a namespace |
| **Azure Kubernetes Service RBAC Writer** | Read/write access to most objects in a namespace |
| **Azure Kubernetes Service RBAC Admin** | Full access within a namespace, including RBAC |
| **Azure Kubernetes Service RBAC Cluster Admin** | Full cluster-wide access |

### Managed Identities

AKS uses **managed identities** to authenticate to Azure services without storing credentials. Two identities are automatically created:

| Identity | Purpose |
|---|---|
| **Cluster (control plane) identity** | Manages cluster infrastructure — load balancers, public IPs, managed disks |
| **Kubelet identity** | Used by nodes to pull images from ACR and access Azure resources |

```bash
# Attach an ACR to AKS using the kubelet managed identity
az aks update \
  --resource-group rg-aks-production \
  --name production-cluster \
  --attach-acr myacr
```

### Workload Identity

**Workload Identity** federates Kubernetes service accounts with Entra ID managed identities, enabling Pods to authenticate to Azure services (Key Vault, Storage, SQL) without secrets.

```bash
# Enable workload identity on the cluster
az aks update \
  --resource-group rg-aks-production \
  --name production-cluster \
  --enable-oidc-issuer \
  --enable-workload-identity

# Get the OIDC issuer URL
export OIDC_ISSUER=$(az aks show \
  --resource-group rg-aks-production \
  --name production-cluster \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

# Create a user-assigned managed identity
az identity create \
  --name id-app-workload \
  --resource-group rg-aks-production

export IDENTITY_CLIENT_ID=$(az identity show \
  --name id-app-workload \
  --resource-group rg-aks-production \
  --query clientId -o tsv)

# Create the federated credential
az identity federated-credential create \
  --name fc-app-workload \
  --identity-name id-app-workload \
  --resource-group rg-aks-production \
  --issuer "$OIDC_ISSUER" \
  --subject "system:serviceaccount:default:sa-app-workload" \
  --audiences "api://AzureADTokenExchange"
```

```yaml
# Kubernetes ServiceAccount annotated for workload identity
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-app-workload
  namespace: default
  annotations:
    azure.workload.identity/client-id: "<IDENTITY_CLIENT_ID>"
  labels:
    azure.workload.identity/use: "true"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-identity
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app-with-identity
  template:
    metadata:
      labels:
        app: app-with-identity
    spec:
      serviceAccountName: sa-app-workload
      containers:
        - name: app
          image: myacr.azurecr.io/app:latest
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
```

## Cluster Autoscaler and KEDA

### Cluster Autoscaler

The **Cluster Autoscaler** adjusts the number of nodes in a node pool based on pending Pod resource requests. It is built into AKS and configured per node pool.

```bash
# Enable autoscaler on an existing node pool
az aks nodepool update \
  --resource-group rg-aks-production \
  --cluster-name production-cluster \
  --name apps \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 20

# Configure autoscaler profile (cluster-wide settings)
az aks update \
  --resource-group rg-aks-production \
  --name production-cluster \
  --cluster-autoscaler-profile \
    scan-interval=10s \
    scale-down-delay-after-add=10m \
    scale-down-unneeded-time=10m \
    max-graceful-termination-sec=600 \
    balance-similar-node-groups=true \
    expander=least-waste
```

### KEDA

**KEDA** (Kubernetes Event-Driven Autoscaling) scales workloads based on external event sources — Azure Service Bus queues, Kafka topics, Prometheus metrics, cron schedules, and more.

```bash
# Install KEDA as an AKS add-on
az aks update \
  --resource-group rg-aks-production \
  --name production-cluster \
  --enable-keda
```

```yaml
# Scale a Deployment based on Azure Service Bus queue depth
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
  namespace: default
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 1
  maxReplicaCount: 50
  pollingInterval: 15
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: orders
        namespace: sb-production
        messageCount: "5"
      authenticationRef:
        name: sb-trigger-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: sb-trigger-auth
  namespace: default
spec:
  podIdentity:
    provider: azure-workload
    identityId: "<IDENTITY_CLIENT_ID>"
```

| Feature | Cluster Autoscaler | KEDA |
|---|---|---|
| **Scales** | Nodes (infrastructure) | Pods (workloads) |
| **Trigger** | Pending Pods with unmet resource requests | External metrics / events |
| **Scale to zero** | Not for system pools | Yes — can scale Deployments to 0 replicas |
| **Use together** | Yes — KEDA scales Pods, Autoscaler adds nodes to fit them |

## Monitoring

### Azure Monitor Container Insights

Container Insights collects metrics, logs, and inventory from AKS clusters using the Azure Monitor agent (AMA).

```bash
# Enable Container Insights on a new cluster
az aks create \
  --resource-group rg-aks-production \
  --name production-cluster \
  --enable-addons monitoring \
  --enable-msi-auth-for-monitoring \
  --generate-ssh-keys

# Enable on an existing cluster
az aks enable-addons \
  --resource-group rg-aks-production \
  --name production-cluster \
  --addons monitoring \
  --enable-msi-auth-for-monitoring

# Configure data collection rules for cost-efficient log collection
az aks update \
  --resource-group rg-aks-production \
  --name production-cluster \
  --data-collection-settings data-collection-settings.json
```

Key data collected:

| Data Type | Source | Destination |
|---|---|---|
| **Performance metrics** | Node and container CPU, memory, disk, network | Azure Monitor Metrics |
| **Container logs** | stdout/stderr from all containers | Log Analytics workspace |
| **Inventory** | Pods, nodes, controllers, services | Log Analytics workspace |
| **Prometheus metrics** | Custom application metrics | Azure Monitor Metrics (via managed Prometheus) |

### Managed Prometheus and Grafana

Azure provides fully managed **Prometheus** and **Grafana** instances for open-source observability on AKS.

```bash
# Create an Azure Monitor workspace (for managed Prometheus)
az monitor account create \
  --name prometheus-aks \
  --resource-group rg-aks-production \
  --location eastus

# Create a managed Grafana instance
az grafana create \
  --name grafana-aks \
  --resource-group rg-aks-production \
  --location eastus

# Link Prometheus metrics to AKS
az aks update \
  --resource-group rg-aks-production \
  --name production-cluster \
  --enable-azure-monitor-metrics \
  --azure-monitor-workspace-resource-id "/subscriptions/<sub>/resourceGroups/rg-aks-production/providers/Microsoft.Monitor/accounts/prometheus-aks" \
  --grafana-resource-id "/subscriptions/<sub>/resourceGroups/rg-aks-production/providers/Microsoft.Dashboard/grafana/grafana-aks"
```

Key metrics to monitor:

| Metric | Source | Alert Threshold |
|---|---|---|
| Node CPU utilization | `node_cpu_seconds_total` | > 80 % sustained |
| Pod restart count | `kube_pod_container_status_restarts_total` | > 5 in 10 min |
| API server latency | `apiserver_request_duration_seconds` | p99 > 1 s |
| Pending Pods | `kube_pod_status_phase{phase="Pending"}` | > 0 for 5 min |
| PVC usage | `kubelet_volume_stats_used_bytes` | > 85 % capacity |
| OOM killed containers | `kube_pod_container_status_last_terminated_reason` | Any OOM event |

## AKS Add-ons and Extensions

AKS supports **add-ons** (Microsoft-managed, first-party) and **extensions** (partner or open-source, via Arc):

| Add-on / Extension | Purpose | Enable Command |
|---|---|---|
| **Monitoring (Container Insights)** | Log and metric collection | `--enable-addons monitoring` |
| **Azure Policy** | Enforce governance with OPA/Gatekeeper | `--enable-addons azure-policy` |
| **KEDA** | Event-driven Pod autoscaling | `--enable-keda` |
| **Azure Key Vault Secrets Provider** | Mount Key Vault secrets as volumes | `--enable-addons azure-keyvault-secrets-provider` |
| **AGIC** | Application Gateway ingress | `--enable-addons ingress-appgw` |
| **Virtual Nodes (ACI)** | Serverless burst via ACI | `--enable-addons virtual-node` |
| **Open Service Mesh** | Service mesh (sidecar-based) | `--enable-addons open-service-mesh` |
| **GitOps (Flux v2)** | Declarative cluster config from Git | `az k8s-extension create --extension-type microsoft.flux` |
| **Dapr** | Distributed Application Runtime | `az k8s-extension create --extension-type microsoft.dapr` |

```bash
# Enable Azure Key Vault Secrets Provider
az aks enable-addons \
  --resource-group rg-aks-production \
  --name production-cluster \
  --addons azure-keyvault-secrets-provider \
  --enable-secret-rotation \
  --rotation-poll-interval 2m

# Enable Azure Policy for AKS
az aks enable-addons \
  --resource-group rg-aks-production \
  --name production-cluster \
  --addons azure-policy
```

## Cost Management

| Strategy | Implementation | Estimated Savings |
|---|---|---|
| **Spot node pools** | Use `--priority Spot` for fault-tolerant workloads | 60–90 % vs On-Demand |
| **Right-sizing** | Deploy VPA to recommend resource requests; use `az aks nodepool scale` | 20–40 % |
| **Scale to zero** | User node pools with `--min-count 0` for non-production | Eliminate idle cost |
| **Reserved Instances** | Commit to 1-yr or 3-yr Azure Reserved VM Instances for baseline nodes | 30–72 % |
| **Azure Savings Plans** | Commit to a fixed hourly compute spend | 15–65 % |
| **Free tier control plane** | Use the Free tier for dev/test clusters | No control plane charge |
| **Start/Stop cluster** | Stop non-production clusters outside business hours | ~65 % (16 hrs/day off) |
| **Namespace quotas** | Set `ResourceQuota` and `LimitRange` per namespace | Prevent resource sprawl |

```bash
# Stop a non-production cluster (deallocates nodes, keeps config)
az aks stop \
  --resource-group rg-aks-dev \
  --name dev-cluster

# Start it back
az aks start \
  --resource-group rg-aks-dev \
  --name dev-cluster

# Check current node utilization
kubectl top nodes

# Find Pods without resource requests (cost risk)
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.containers[].resources.requests == null) |
  "\(.metadata.namespace)/\(.metadata.name)"'
```

## Best Practices

### Security

- **Enable Microsoft Entra ID integration** and Azure RBAC — avoid local accounts (`--disable-local-accounts`)
- **Use private clusters** for production — disable public API server access
- **Use Workload Identity** — never store Azure credentials in Kubernetes Secrets
- **Enable Defender for Containers** for runtime threat detection and vulnerability scanning
- **Scan container images** with Microsoft Defender for Container Registries or Trivy
- **Apply Pod Security Standards** using Azure Policy or Kubernetes admission controllers
- **Encrypt secrets with Azure Key Vault** via the Secrets Store CSI driver

### Reliability

- **Spread across 3 Availability Zones** using `--zones 1 2 3` and pod topology spread constraints
- **Set resource requests and limits** on all containers to avoid OOM kills and noisy neighbors
- **Use Pod Disruption Budgets (PDBs)** for critical workloads during upgrades and node drains
- **Use the Standard or Premium tier** for uptime SLA on the control plane
- **Enable cluster autoscaler** to handle variable load without manual intervention
- **Run at least 2 replicas** of critical workloads with anti-affinity across zones

### Operations

- **Automate upgrades** — keep control plane and node pools within one minor version; use maintenance windows
- **Tag all resources** with `Environment`, `Team`, and `CostCenter` for cost allocation
- **Use GitOps** (Flux v2 or Argo CD) for declarative cluster configuration
- **Store Terraform state remotely** in Azure Storage with state locking
- **Test upgrades in a staging cluster** before rolling out to production
- **Enable diagnostic settings** to stream control plane logs to Log Analytics

### Networking

- **Use Azure CNI Overlay with Cilium** for modern, scalable networking with eBPF
- **Size subnets generously** — account for node surge during upgrades
- **Use Network Policies** (Cilium or Azure) to enforce east-west traffic rules
- **Deploy internal load balancers** for services that should not be publicly exposed
- **Use Private Endpoints** for Azure PaaS services to keep traffic on the Azure backbone

## Next Steps

- [Kubernetes Services](01-SERVICES.md) — revisit service types and traffic routing
- **Helm and Package Management** — managing application charts on AKS
- **CI/CD Pipelines** — deploy to AKS from Azure DevOps, GitHub Actions, or Argo CD
- **Service Mesh** — Istio or Open Service Mesh for advanced traffic management
- **Multi-Cluster** — Azure Fleet Manager, federation, and multi-region AKS patterns
