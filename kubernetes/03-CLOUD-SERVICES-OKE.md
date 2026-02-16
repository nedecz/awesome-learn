# Oracle OKE — Oracle Kubernetes Engine

## Table of Contents

1. [Overview](#overview)
2. [OKE Architecture](#oke-architecture)
   - [Control Plane](#control-plane)
   - [Data Plane](#data-plane)
3. [Prerequisites and OCI Setup](#prerequisites-and-oci-setup)
   - [Required Tools](#required-tools)
   - [Compartments and Policies](#compartments-and-policies)
4. [Creating an OKE Cluster](#creating-an-oke-cluster)
   - [Using OCI Console](#using-oci-console)
   - [Using OCI CLI](#using-oci-cli)
   - [Using Terraform](#using-terraform)
5. [Networking](#networking)
   - [VCN Design](#vcn-design)
   - [Subnets](#subnets)
   - [Security Lists and Network Security Groups](#security-lists-and-network-security-groups)
   - [VCN-Native Pod Networking](#vcn-native-pod-networking)
6. [Node Pools](#node-pools)
   - [Managed Nodes](#managed-nodes)
   - [Virtual Nodes](#virtual-nodes)
   - [Node Pool Configuration](#node-pool-configuration)
7. [Storage](#storage)
   - [OCI Block Volume CSI Driver](#oci-block-volume-csi-driver)
   - [OCI File Storage CSI Driver](#oci-file-storage-csi-driver)
   - [StorageClasses](#storageclasses)
8. [Load Balancing](#load-balancing)
   - [OCI Load Balancer](#oci-load-balancer)
   - [OCI Flexible Load Balancer](#oci-flexible-load-balancer)
   - [Ingress Controller](#ingress-controller)
9. [Authentication and Authorization](#authentication-and-authorization)
   - [OCI IAM and RBAC](#oci-iam-and-rbac)
   - [Instance Principals](#instance-principals)
   - [Workload Identity](#workload-identity)
10. [Cluster Add-ons and Management](#cluster-add-ons-and-management)
11. [Monitoring](#monitoring)
    - [OCI Monitoring and Alarms](#oci-monitoring-and-alarms)
    - [OCI Logging](#oci-logging)
    - [Prometheus and Grafana](#prometheus-and-grafana)
12. [Best Practices](#best-practices)
13. [Next Steps](#next-steps)

## Overview

This document covers **Oracle Kubernetes Engine (OKE)** — Oracle Cloud Infrastructure's managed Kubernetes service that provisions and manages the Kubernetes control plane and, optionally, the worker nodes. OKE integrates natively with OCI networking, identity, storage, and observability services, and is certified conformant by the CNCF.

### Target Audience

- DevOps and platform engineers deploying workloads on Oracle Cloud Infrastructure
- Developers building and shipping containerized applications to OKE
- SREs operating production Kubernetes clusters on OCI
- Architects evaluating managed Kubernetes options across cloud providers

### Scope

- OKE architecture, control plane, managed nodes, and virtual nodes
- Cluster creation with OCI Console, OCI CLI, and Terraform
- Networking (VCN-Native Pod Networking), storage (Block Volume / File Storage CSI), and load balancing
- OCI IAM integration, instance principals, workload identity, and RBAC
- Cluster add-ons, monitoring, and operational best practices

## OKE Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                       OCI Region                                    │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │            OKE Control Plane (Oracle-managed)               │   │
│  │                                                              │   │
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐                │   │
│  │  │ API      │   │ API      │   │ API      │   HA across    │   │
│  │  │ Server   │   │ Server   │   │ Server   │   3 ADs / FDs  │   │
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
│  │  AD-1 / FD-1          AD-2 / FD-2          AD-3 / FD-3      │   │
│  │  ┌──────────┐        ┌──────────┐        ┌──────────┐      │   │
│  │  │ Managed  │        │ Managed  │        │ Virtual  │      │   │
│  │  │ Node     │        │ Node     │        │ Node     │      │   │
│  │  │ ┌──┐┌──┐ │        │ ┌──┐┌──┐ │        │ ┌──┐     │      │   │
│  │  │ │P1││P2│ │        │ │P3││P4│ │        │ │P5│     │      │   │
│  │  │ └──┘└──┘ │        │ └──┘└──┘ │        │ └──┘     │      │   │
│  │  └──────────┘        └──────────┘        └──────────┘      │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Control Plane

Oracle fully manages the OKE control plane. You do **not** provision, patch, or scale API servers or etcd. Key characteristics:

| Aspect | Detail |
|---|---|
| **API servers** | Distributed across Availability Domains (ADs) or Fault Domains (FDs) |
| **etcd** | Managed, encrypted, backed up automatically |
| **SLA** | 99.95 % uptime (Financial SLA) |
| **Upgrades** | Oracle provides in-place Kubernetes version upgrades |
| **Endpoint access** | Public, private, or both (configurable) |
| **Cost** | Control plane is **free** — you pay only for worker nodes |

### Data Plane

You choose how to run your workloads:

| Option | Description | Use Case |
|---|---|---|
| **Managed nodes** | You control the compute shape and image; OKE manages lifecycle | General-purpose, production workloads |
| **Virtual nodes** | Serverless — Oracle provisions and manages infrastructure per Pod | Burst workloads, low-ops environments |

## Prerequisites and OCI Setup

### Required Tools

Install the following CLI tools before working with OKE:

```bash
# OCI CLI
bash -c "$(curl -L https://raw.githubusercontent.com/oracle/oci-cli/master/scripts/install/install.sh)"

# Configure the CLI with your tenancy details
oci setup config

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Verify installations
oci --version
kubectl version --client
```

### Compartments and Policies

OKE requires an OCI compartment and IAM policies that grant the OKE service permission to manage resources.

**1. Create a compartment** for your Kubernetes resources:

```bash
oci iam compartment create \
  --compartment-id <tenancy-ocid> \
  --name kubernetes \
  --description "Compartment for OKE clusters and related resources"
```

**2. Create the required IAM policies** — the OKE service must be able to manage networking, compute, and other resources:

```bash
# Policy statements for OKE
oci iam policy create \
  --compartment-id <tenancy-ocid> \
  --name oke-service-policy \
  --description "Allow OKE to manage cluster resources" \
  --statements '[
    "Allow service OKE to manage all-resources in compartment kubernetes",
    "Allow group oke-admins to manage cluster-family in compartment kubernetes",
    "Allow group oke-admins to manage virtual-network-family in compartment kubernetes",
    "Allow group oke-admins to manage instance-family in compartment kubernetes",
    "Allow group oke-admins to use subnets in compartment kubernetes",
    "Allow group oke-admins to use vnics in compartment kubernetes",
    "Allow group oke-admins to inspect compartments in tenancy"
  ]'
```

## Creating an OKE Cluster

### Using OCI Console

Step-by-step process through the OCI web console:

1. Navigate to **Developer Services → Kubernetes Clusters (OKE)**
2. Click **Create Cluster**
3. Choose a workflow:
   - **Quick Create** — OCI provisions the VCN, subnets, security rules, and node pool automatically
   - **Custom Create** — you select an existing VCN, subnets, and configure every option manually
4. Configure cluster settings:
   - **Name:** `production-cluster`
   - **Compartment:** `kubernetes`
   - **Kubernetes Version:** select the latest supported version (e.g., `v1.31.1`)
   - **API Endpoint:** Public or Private
   - **Kubernetes Worker Nodes:** Public or Private subnets
5. Configure the node pool:
   - **Shape:** `VM.Standard.E4.Flex` (2 OCPUs, 16 GB RAM)
   - **Number of nodes:** 3
   - **Placement:** distribute across Availability Domains or Fault Domains
6. Review and click **Create Cluster**
7. Once the cluster is **Active**, click **Access Cluster** and follow the `oci ce cluster create-kubeconfig` command

### Using OCI CLI

```bash
# Step 1 — Create the cluster
oci ce cluster create \
  --compartment-id <compartment-ocid> \
  --name production-cluster \
  --kubernetes-version v1.31.1 \
  --vcn-id <vcn-ocid> \
  --service-lb-subnet-ids '["<public-subnet-ocid>"]' \
  --endpoint-subnet-id <endpoint-subnet-ocid> \
  --endpoint-public-ip-enabled true

# Step 2 — Wait for the cluster to become ACTIVE
oci ce cluster get \
  --cluster-id <cluster-ocid> \
  --query 'data."lifecycle-state"' \
  --raw-output

# Step 3 — Create a node pool
oci ce node-pool create \
  --compartment-id <compartment-ocid> \
  --cluster-id <cluster-ocid> \
  --name standard-nodes \
  --kubernetes-version v1.31.1 \
  --node-shape VM.Standard.E4.Flex \
  --node-shape-config '{"ocpus": 2, "memoryInGBs": 16}' \
  --node-config-details \
    '{"size": 3,
      "placementConfigs": [
        {"availabilityDomain": "AD-1", "subnetId": "<worker-subnet-ocid>"},
        {"availabilityDomain": "AD-2", "subnetId": "<worker-subnet-ocid>"},
        {"availabilityDomain": "AD-3", "subnetId": "<worker-subnet-ocid>"}
      ]}'

# Step 4 — Generate kubeconfig
oci ce cluster create-kubeconfig \
  --cluster-id <cluster-ocid> \
  --file $HOME/.kube/config \
  --region us-ashburn-1 \
  --token-version 2.0.0 \
  --kube-endpoint PUBLIC_ENDPOINT

# Step 5 — Verify nodes
kubectl get nodes
```

### Using Terraform

Terraform provides a declarative, version-controlled approach to infrastructure. The example below uses the OCI provider.

```hcl
# main.tf
terraform {
  required_version = ">= 1.5"
  required_providers {
    oci = {
      source  = "oracle/oci"
      version = "~> 6.0"
    }
  }
}

provider "oci" {
  region = var.region
}

# ---------- VCN ----------
resource "oci_core_vcn" "oke_vcn" {
  compartment_id = var.compartment_id
  display_name   = "${var.cluster_name}-vcn"
  cidr_blocks    = ["10.0.0.0/16"]
  dns_label      = "okecluster"
}

resource "oci_core_internet_gateway" "igw" {
  compartment_id = var.compartment_id
  vcn_id         = oci_core_vcn.oke_vcn.id
  display_name   = "internet-gateway"
}

resource "oci_core_nat_gateway" "natgw" {
  compartment_id = var.compartment_id
  vcn_id         = oci_core_vcn.oke_vcn.id
  display_name   = "nat-gateway"
}

resource "oci_core_route_table" "public_rt" {
  compartment_id = var.compartment_id
  vcn_id         = oci_core_vcn.oke_vcn.id
  display_name   = "public-route-table"

  route_rules {
    destination       = "0.0.0.0/0"
    network_entity_id = oci_core_internet_gateway.igw.id
  }
}

resource "oci_core_route_table" "private_rt" {
  compartment_id = var.compartment_id
  vcn_id         = oci_core_vcn.oke_vcn.id
  display_name   = "private-route-table"

  route_rules {
    destination       = "0.0.0.0/0"
    network_entity_id = oci_core_nat_gateway.natgw.id
  }
}

resource "oci_core_subnet" "api_endpoint_subnet" {
  compartment_id = var.compartment_id
  vcn_id         = oci_core_vcn.oke_vcn.id
  display_name   = "api-endpoint-subnet"
  cidr_block     = "10.0.0.0/28"
  route_table_id = oci_core_route_table.public_rt.id
  dns_label      = "apiendpoint"
}

resource "oci_core_subnet" "worker_subnet" {
  compartment_id             = var.compartment_id
  vcn_id                     = oci_core_vcn.oke_vcn.id
  display_name               = "worker-subnet"
  cidr_block                 = "10.0.1.0/24"
  route_table_id             = oci_core_route_table.private_rt.id
  prohibit_public_ip_on_vnic = true
  dns_label                  = "workers"
}

resource "oci_core_subnet" "service_lb_subnet" {
  compartment_id = var.compartment_id
  vcn_id         = oci_core_vcn.oke_vcn.id
  display_name   = "service-lb-subnet"
  cidr_block     = "10.0.2.0/24"
  route_table_id = oci_core_route_table.public_rt.id
  dns_label      = "servicelb"
}

# ---------- OKE Cluster ----------
resource "oci_containerengine_cluster" "oke" {
  compartment_id     = var.compartment_id
  kubernetes_version = "v1.31.1"
  name               = var.cluster_name
  vcn_id             = oci_core_vcn.oke_vcn.id

  endpoint_config {
    is_public_ip_enabled = true
    subnet_id            = oci_core_subnet.api_endpoint_subnet.id
  }

  options {
    service_lb_subnet_ids = [oci_core_subnet.service_lb_subnet.id]
  }
}

# ---------- Node Pool ----------
resource "oci_containerengine_node_pool" "standard" {
  compartment_id     = var.compartment_id
  cluster_id         = oci_containerengine_cluster.oke.id
  kubernetes_version = "v1.31.1"
  name               = "standard-nodes"

  node_shape = "VM.Standard.E4.Flex"
  node_shape_config {
    ocpus         = 2
    memory_in_gbs = 16
  }

  node_config_details {
    size = 3

    placement_configs {
      availability_domain = data.oci_identity_availability_domains.ads.availability_domains[0].name
      subnet_id           = oci_core_subnet.worker_subnet.id
    }
    placement_configs {
      availability_domain = data.oci_identity_availability_domains.ads.availability_domains[1].name
      subnet_id           = oci_core_subnet.worker_subnet.id
    }
  }

  node_source_details {
    source_type = "IMAGE"
    image_id    = var.node_image_id
  }
}

data "oci_identity_availability_domains" "ads" {
  compartment_id = var.compartment_id
}

# ---------- Variables ----------
variable "region" {
  default = "us-ashburn-1"
}

variable "compartment_id" {
  description = "OCID of the compartment for OKE resources"
}

variable "cluster_name" {
  default = "production-cluster"
}

variable "node_image_id" {
  description = "OCID of the OKE node image"
}

# ---------- Outputs ----------
output "cluster_id" {
  value = oci_containerengine_cluster.oke.id
}

output "cluster_endpoint" {
  value = oci_containerengine_cluster.oke.endpoints[0].public_endpoint
}
```

```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan

# Generate kubeconfig
oci ce cluster create-kubeconfig \
  --cluster-id $(terraform output -raw cluster_id) \
  --region us-ashburn-1 \
  --token-version 2.0.0 \
  --kube-endpoint PUBLIC_ENDPOINT
```

## Networking

### VCN Design

OKE clusters run inside an OCI Virtual Cloud Network (VCN). A well-designed VCN separates the API endpoint, worker nodes, load balancers, and optionally Pods into distinct subnets.

```
┌─────────────────────────── VCN 10.0.0.0/16 ───────────────────────────┐
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ API Endpoint Subnet  10.0.0.0/28  (public or private)       │      │
│  │  ┌───────────────────┐                                      │      │
│  │  │ Kubernetes API    │                                      │      │
│  │  └───────────────────┘                                      │      │
│  └──────────────────────────────────────────────────────────────┘      │
│                                                                        │
│  AD-1                     AD-2                     AD-3                │
│  ┌────────────────┐      ┌────────────────┐      ┌────────────────┐   │
│  │ Worker Subnet  │      │ Worker Subnet  │      │ Worker Subnet  │   │
│  │ 10.0.1.0/24    │      │ 10.0.1.0/24    │      │ 10.0.1.0/24    │   │
│  │ (private)      │      │ (private)      │      │ (private)      │   │
│  │ ┌────┐ ┌────┐  │      │ ┌────┐ ┌────┐  │      │ ┌────┐ ┌────┐  │   │
│  │ │Node│ │Node│  │      │ │Node│ │Node│  │      │ │Node│ │Node│  │   │
│  │ └────┘ └────┘  │      │ └────┘ └────┘  │      │ └────┘ └────┘  │   │
│  └────────────────┘      └────────────────┘      └────────────────┘   │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ Service LB Subnet  10.0.2.0/24  (public)                    │      │
│  │  ┌──────────┐  ┌──────────┐                                 │      │
│  │  │   LB     │  │   LB     │                                 │      │
│  │  └──────────┘  └──────────┘                                 │      │
│  └──────────────────────────────────────────────────────────────┘      │
│                                                                        │
│  ┌──────────┐  ┌──────────┐                                           │
│  │ IGW      │  │ NAT GW   │                                           │
│  └──────────┘  └──────────┘                                           │
└────────────────────────────────────────────────────────────────────────┘
```

### Subnets

OKE requires at least three subnets:

| Subnet | Purpose | Visibility | Typical CIDR |
|---|---|---|---|
| **API Endpoint** | Kubernetes API server endpoint | Public or Private | `10.0.0.0/28` |
| **Worker Nodes** | Compute instances running Pods | Private | `10.0.1.0/24` |
| **Service LBs** | OCI Load Balancers for Services | Public | `10.0.2.0/24` |
| **Pod Subnet** (optional) | Dedicated subnet for VCN-Native Pod Networking | Private | `10.0.16.0/20` |

> **Tip:** Always place worker nodes in private subnets. Use a NAT gateway for outbound internet access and expose services through load balancers in public subnets.

### Security Lists and Network Security Groups

OKE uses **Security Lists** (stateful firewall rules on subnets) and **Network Security Groups (NSGs)** for fine-grained control.

Key rules for the **worker node** subnet:

| Direction | Port | Source / Destination | Purpose |
|---|---|---|---|
| Ingress | 6443 | API endpoint subnet | API server → kubelet |
| Ingress | 10250 | API endpoint subnet | API server → kubelet (metrics) |
| Ingress | 12250 | Worker subnet (self) | ICMP / node-to-node |
| Ingress | 30000–32767 | Service LB subnet | NodePort traffic |
| Egress | 6443 | API endpoint subnet | kubelet → API server |
| Egress | All | 0.0.0.0/0 via NAT GW | Container image pulls, external APIs |

Key rules for the **API endpoint** subnet:

| Direction | Port | Source / Destination | Purpose |
|---|---|---|---|
| Ingress | 6443 | 0.0.0.0/0 or admin CIDR | kubectl / client access |
| Ingress | 6443 | Worker subnet | kubelet → API server |
| Egress | 10250 | Worker subnet | API server → kubelet |
| Egress | 443 | OCI services | OCI API calls |

### VCN-Native Pod Networking

OKE supports **VCN-Native Pod Networking**, which assigns VCN IP addresses directly to Pods (similar to AWS VPC CNI). Each Pod gets a real IP from a dedicated Pod subnet, making Pods routable within the VCN without overlays.

```
┌────────── Managed Node (VM.Standard.E4.Flex) ──────────┐
│  Primary VNIC (worker subnet)                           │
│    IP: 10.0.1.10                                        │
│                                                         │
│  Pod VNICs (pod subnet — 10.0.16.0/20)                  │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐               │
│  │ Pod  │  │ Pod  │  │ Pod  │  │ Pod  │               │
│  │.16.5 │  │.16.6 │  │.16.7 │  │.16.8 │               │
│  └──────┘  └──────┘  └──────┘  └──────┘               │
└─────────────────────────────────────────────────────────┘
```

Enable VCN-Native Pod Networking when creating a cluster by selecting the **OCI VCN-Native Pod Networking CNI** and providing a Pod subnet:

```bash
oci ce cluster create \
  --compartment-id <compartment-ocid> \
  --name production-cluster \
  --kubernetes-version v1.31.1 \
  --vcn-id <vcn-ocid> \
  --service-lb-subnet-ids '["<lb-subnet-ocid>"]' \
  --endpoint-subnet-id <endpoint-subnet-ocid> \
  --cluster-pod-network-options '[{"cniType": "OCI_VCN_IP_NATIVE"}]'
```

## Node Pools

### Managed Nodes

Managed nodes run on OCI Compute instances that you configure. OKE handles joining nodes to the cluster and supports cordon/drain during upgrades.

```bash
# Create a managed node pool
oci ce node-pool create \
  --compartment-id <compartment-ocid> \
  --cluster-id <cluster-ocid> \
  --name app-nodes \
  --kubernetes-version v1.31.1 \
  --node-shape VM.Standard.E4.Flex \
  --node-shape-config '{"ocpus": 4, "memoryInGBs": 64}' \
  --node-config-details \
    '{"size": 3,
      "placementConfigs": [
        {"availabilityDomain": "AD-1", "subnetId": "<worker-subnet-ocid>"}
      ]}'

# List node pools
oci ce node-pool list --compartment-id <compartment-ocid> --cluster-id <cluster-ocid>

# Scale a node pool
oci ce node-pool update \
  --node-pool-id <node-pool-ocid> \
  --node-config-details '{"size": 5}'

# Upgrade a node pool to a new Kubernetes version
oci ce node-pool update \
  --node-pool-id <node-pool-ocid> \
  --kubernetes-version v1.32.1
```

### Virtual Nodes

**Virtual nodes** provide a serverless experience. Oracle manages the underlying infrastructure and you are charged per Pod based on resource requests. No patching, scaling, or managing compute instances.

```bash
# Create a virtual node pool
oci ce virtual-node-pool create \
  --compartment-id <compartment-ocid> \
  --cluster-id <cluster-ocid> \
  --display-name virtual-pool \
  --kubernetes-version v1.31.1 \
  --size 1 \
  --pod-configuration \
    '{"subnetId": "<pod-subnet-ocid>",
      "shape": "Pod.Standard.E4.Flex",
      "nsgIds": ["<nsg-ocid>"]}'
```

**Virtual node limitations:**

- No DaemonSets (Pods run in isolated environments)
- No HostPath or local persistent volumes
- Limited to specific container shapes
- Pod resource requests define billing

### Node Pool Configuration

Common configuration options for managed node pools:

| Option | Description | Example |
|---|---|---|
| **Shape** | Compute instance type | `VM.Standard.E4.Flex`, `VM.Standard.A1.Flex` (Arm) |
| **Node image** | OS image for the nodes | OKE-provided Oracle Linux images |
| **Cloud-init** | Custom startup scripts | Install monitoring agents, configure mounts |
| **Labels** | Kubernetes labels applied to nodes | `role=applications`, `tier=frontend` |
| **Taints** | Kubernetes taints to control scheduling | `dedicated=gpu:NoSchedule` |
| **Placement** | AD / FD placement configuration | Spread across fault domains |

## Storage

### OCI Block Volume CSI Driver

The **OCI Block Volume CSI driver** dynamically provisions block storage volumes and attaches them to Pods. It is installed by default on OKE clusters.

```bash
# Verify the CSI driver is running
kubectl get pods -n kube-system -l app=csi-oci-node
```

### OCI File Storage CSI Driver

The **OCI File Storage (FSS) CSI driver** provides shared NFS-based file systems accessible by multiple Pods across Availability Domains.

```bash
# Install the File Storage CSI driver (if not already installed)
kubectl apply -f https://raw.githubusercontent.com/oracle/oci-cloud-controller-manager/master/manifests/provider-config-instance-principals-example.yaml
```

### StorageClasses

```yaml
# block-storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: oci-bv
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: blockvolume.csi.oraclecloud.com
parameters:
  vpusPerGB: "20"        # 10=Balanced, 20=Higher Performance
  attachment-type: "paravirtualized"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
---
# file-storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: oci-fss
provisioner: fss.csi.oraclecloud.com
parameters:
  availabilityDomain: "AD-1"
  mountTargetOcid: "ocid1.mounttarget.oc1..."
  compartmentOcid: "ocid1.compartment.oc1..."
  exportPath: "/oke-fss"
  exportOptions: '[{"source":"10.0.0.0/16","requirePrivilegedSourcePort":false,"access":"READ_WRITE","identitySquash":"NONE"}]'
reclaimPolicy: Delete
```

```yaml
# Example PersistentVolumeClaim using block storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  storageClassName: oci-bv
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

```bash
kubectl apply -f block-storage-class.yaml
kubectl apply -f file-storage-class.yaml
```

## Load Balancing

### OCI Load Balancer

When you create a Kubernetes Service of type `LoadBalancer`, OKE automatically provisions an **OCI Load Balancer**. The OCI Cloud Controller Manager handles the lifecycle.

```yaml
# oci-loadbalancer-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: app-lb
  annotations:
    oci.oraclecloud.com/load-balancer-type: "lb"
    service.beta.kubernetes.io/oci-load-balancer-shape: "flexible"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-min: "10"
    service.beta.kubernetes.io/oci-load-balancer-shape-flex-max: "100"
    service.beta.kubernetes.io/oci-load-balancer-subnet1: "<public-subnet-ocid>"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 443
      targetPort: 8443
      protocol: TCP
```

### OCI Flexible Load Balancer

The **Flexible Load Balancer** lets you specify a bandwidth range rather than a fixed shape, scaling automatically based on traffic.

| Annotation | Description | Example |
|---|---|---|
| `oci-load-balancer-shape` | Shape type | `flexible` |
| `oci-load-balancer-shape-flex-min` | Minimum bandwidth (Mbps) | `10` |
| `oci-load-balancer-shape-flex-max` | Maximum bandwidth (Mbps) | `8192` |
| `oci-load-balancer-internal` | Create a private load balancer | `true` |
| `oci-load-balancer-subnet1` | Subnet OCID for the LB | OCID string |
| `oci-load-balancer-security-list-management-mode` | Security list management | `All`, `Frontend`, `None` |

### Ingress Controller

Deploy an **NGINX Ingress Controller** backed by an OCI Load Balancer for Layer 7 routing:

```bash
# Install NGINX Ingress Controller via Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."oci\.oraclecloud\.com/load-balancer-type"="lb" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/oci-load-balancer-shape"="flexible" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/oci-load-balancer-shape-flex-min"="10" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/oci-load-balancer-shape-flex-max"="100"
```

```yaml
# ingress-example.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
```

```
                 Internet
                    │
           ┌───────▼───────┐
           │  OCI Load     │     ← Flexible LB provisioned by CCM
           │  Balancer     │
           └───┬───┬───┬───┘
               │   │   │
          ┌────▼┐┌─▼──┐┌▼────┐
          │ Pod ││ Pod ││ Pod │   ← NGINX Ingress Controller Pods
          └─────┘└────┘└─────┘
               │   │   │
          ┌────▼┐┌─▼──┐┌▼────┐
          │ App ││ App ││ App │   ← Backend application Pods
          └─────┘└────┘└─────┘
```

## Authentication and Authorization

### OCI IAM and RBAC

OKE integrates with **OCI Identity and Access Management (IAM)** for cluster access. Users authenticate via the OCI CLI, and authorization is determined by both OCI IAM policies and Kubernetes RBAC.

```bash
# Generate kubeconfig (uses OCI token for authentication)
oci ce cluster create-kubeconfig \
  --cluster-id <cluster-ocid> \
  --file $HOME/.kube/config \
  --region us-ashburn-1 \
  --token-version 2.0.0 \
  --kube-endpoint PUBLIC_ENDPOINT
```

Map OCI IAM groups to Kubernetes RBAC with ClusterRoleBindings:

```yaml
# rbac-oci-admins.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: oci-cluster-admins
subjects:
  - kind: Group
    name: <oci-group-ocid>
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### Instance Principals

**Instance principals** allow compute instances (including OKE worker nodes) to call OCI APIs without storing credentials. The OCI Cloud Controller Manager and CSI drivers use instance principals by default.

Required dynamic group and policy:

```bash
# Create a dynamic group matching OKE worker nodes
oci iam dynamic-group create \
  --compartment-id <tenancy-ocid> \
  --name oke-worker-nodes \
  --description "OKE worker node instances" \
  --matching-rule "ALL {instance.compartment.id = '<compartment-ocid>'}"

# Grant permissions to the dynamic group
oci iam policy create \
  --compartment-id <compartment-ocid> \
  --name oke-instance-principal-policy \
  --description "Allow OKE nodes to manage OCI resources" \
  --statements '[
    "Allow dynamic-group oke-worker-nodes to manage volumes in compartment kubernetes",
    "Allow dynamic-group oke-worker-nodes to manage load-balancers in compartment kubernetes",
    "Allow dynamic-group oke-worker-nodes to use virtual-network-family in compartment kubernetes",
    "Allow dynamic-group oke-worker-nodes to manage file-systems in compartment kubernetes"
  ]'
```

### Workload Identity

**OKE Workload Identity** allows Kubernetes service accounts to authenticate directly with OCI services. Pods can access OCI resources (Object Storage, Autonomous Database, etc.) using their Kubernetes identity — no static credentials needed.

```bash
# Create a workload identity policy
oci iam policy create \
  --compartment-id <compartment-ocid> \
  --name workload-identity-policy \
  --description "Allow Kubernetes workloads to access OCI services" \
  --statements '[
    "Allow any-user to read objects in compartment kubernetes where all {request.principal.type = '"'"'workload'"'"', request.principal.namespace = '"'"'default'"'"', request.principal.service_account = '"'"'app-sa'"'"', request.principal.cluster_id = '"'"'<cluster-ocid>'"'"'}"
  ]'
```

```yaml
# Service account for workload identity
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: default
---
apiVersion: v1
kind: Pod
metadata:
  name: oci-sdk-app
  namespace: default
spec:
  serviceAccountName: app-sa
  containers:
    - name: app
      image: iad.ocir.io/my-namespace/my-app:latest
      env:
        - name: OCI_RESOURCE_PRINCIPAL_VERSION
          value: "2.2"
```

## Cluster Add-ons and Management

OKE supports **cluster add-ons** — optional, Oracle-managed components that can be enabled on a cluster. Add-ons are kept up-to-date by Oracle.

| Add-on | Purpose |
|---|---|
| **CoreDNS** | Cluster DNS resolution |
| **kube-proxy** | Kubernetes network proxy |
| **OCI VCN-Native Pod Networking CNI** | Pod networking via VCN IPs |
| **Kubernetes Dashboard** | Web-based cluster UI |
| **Certificate Manager (cert-manager)** | TLS certificate lifecycle |
| **Tiller (legacy)** | Helm v2 server (deprecated) |

```bash
# List available add-ons for a cluster
oci ce cluster list-addons --cluster-id <cluster-ocid>

# Enable an add-on
oci ce cluster install-addon \
  --cluster-id <cluster-ocid> \
  --addon-name CertManager \
  --version v1.16.1

# Disable an add-on
oci ce cluster disable-addon \
  --cluster-id <cluster-ocid> \
  --addon-name KubernetesDashboard \
  --is-remove-existing-add-on true
```

**Cluster management operations:**

```bash
# Upgrade the control plane Kubernetes version
oci ce cluster update \
  --cluster-id <cluster-ocid> \
  --kubernetes-version v1.32.1

# Rotate cluster credentials
oci ce cluster rotate-credentials --cluster-id <cluster-ocid>

# Delete a cluster
oci ce cluster delete --cluster-id <cluster-ocid>
```

## Monitoring

### OCI Monitoring and Alarms

OKE emits metrics to **OCI Monitoring** automatically. You can create alarms based on these metrics.

Key OKE metrics in the `oci_oke` namespace:

| Metric | Description | Alert Threshold |
|---|---|---|
| `NodeState` | Health state of worker nodes | != `ACTIVE` |
| `NodePoolState` | Health of the node pool | != `ACTIVE` |
| `UnschedulablePods` | Pods that cannot be scheduled | > 0 for 5 min |
| `APIServerRequestCount` | Rate of API server requests | Anomaly detection |
| `APIServerRequestLatency` | Latency of API server requests | p99 > 1 s |

```bash
# Create an alarm for unhealthy nodes
oci monitoring alarm create \
  --compartment-id <compartment-ocid> \
  --display-name "OKE Unhealthy Nodes" \
  --namespace oci_oke \
  --query 'NodeState[1m]{resourceId = "<cluster-ocid>"}.count() > 0' \
  --severity CRITICAL \
  --destinations '["<notification-topic-ocid>"]' \
  --is-enabled true
```

### OCI Logging

Enable **OCI Logging** to capture API server audit logs, worker node logs, and container stdout/stderr.

```bash
# Create a log group
oci logging log-group create \
  --compartment-id <compartment-ocid> \
  --display-name oke-logs

# Enable OKE API server audit logging
oci logging log create \
  --log-group-id <log-group-ocid> \
  --display-name oke-api-audit \
  --log-type SERVICE \
  --configuration '{
    "source": {
      "service": "oke",
      "resource": "<cluster-ocid>",
      "category": "kube-apiserver-audit"
    },
    "compartmentId": "<compartment-ocid>"
  }' \
  --is-enabled true
```

### Prometheus and Grafana

For open-source observability, deploy Prometheus and Grafana on your OKE cluster:

```bash
# Install Prometheus and Grafana via Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=15d \
  --set grafana.adminPassword=changeme
```

Key metrics to monitor:

| Metric | Source | Alert Threshold |
|---|---|---|
| Node CPU utilization | `node_cpu_seconds_total` | > 80 % sustained |
| Pod restart count | `kube_pod_container_status_restarts_total` | > 5 in 10 min |
| API server latency | `apiserver_request_duration_seconds` | p99 > 1 s |
| Pending Pods | `kube_pod_status_phase{phase="Pending"}` | > 0 for 5 min |
| PVC usage | `kubelet_volume_stats_used_bytes` | > 85 % capacity |

## Best Practices

### Security

- **Use private API endpoints** and restrict public access to known CIDRs
- **Enable workload identity** — avoid static credentials or config-file-based authentication for OCI API access
- **Use Network Security Groups** instead of Security Lists for fine-grained, per-resource firewall rules
- **Enable API server audit logging** via OCI Logging and monitor with OCI Events
- **Scan container images** with Oracle Cloud Infrastructure Vulnerability Scanning or a third-party tool
- **Apply Pod Security Standards** using Kubernetes admission controllers
- **Encrypt secrets at rest** using OCI Vault integration

### Reliability

- **Spread across 3 Fault Domains** (or Availability Domains) using pod topology spread constraints
- **Set resource requests and limits** on all containers to avoid OOM kills and noisy neighbors
- **Use Pod Disruption Budgets (PDBs)** for critical workloads during node pool upgrades
- **Run CoreDNS with anti-affinity** across nodes for DNS resilience
- **Use multiple node pools** — separate system components from application workloads

### Operations

- **Automate upgrades** — keep control plane and node pools within one minor version
- **Tag all resources** with `Environment`, `Team`, and `CostCenter` for cost tracking via OCI Cost Analysis
- **Use GitOps** (Flux or Argo CD) for declarative cluster configuration
- **Store Terraform state remotely** in OCI Object Storage with state locking
- **Test upgrades in a staging cluster** before rolling out to production

### Networking

- **Use VCN-Native Pod Networking** for workloads requiring direct VCN IP reachability
- **Use NSGs over Security Lists** for portability and per-resource rules
- **Deploy the NGINX Ingress Controller** or OCI Native Ingress Controller for Layer 7 routing
- **Reserve IP space** — plan your VCN CIDR to accommodate growth in nodes and Pods

## Next Steps

- [Kubernetes Services](01-SERVICES.md) — revisit service types and traffic routing
- **Helm and Package Management** — managing application charts on OKE
- **CI/CD Pipelines** — deploy to OKE from OCI DevOps, GitHub Actions, or Argo CD
- **Service Mesh** — Istio or Oracle Service Mesh for advanced traffic management
- **Multi-Cluster** — federation, failover, and multi-region OKE patterns
