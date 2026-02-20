# AWS EKS — Elastic Kubernetes Service

## Table of Contents

1. [Overview](#overview)
2. [EKS Architecture](#eks-architecture)
   - [Control Plane](#control-plane)
   - [Data Plane](#data-plane)
3. [Prerequisites and IAM Setup](#prerequisites-and-iam-setup)
   - [Required Tools](#required-tools)
   - [IAM Roles and Policies](#iam-roles-and-policies)
4. [Creating an EKS Cluster](#creating-an-eks-cluster)
   - [Using eksctl](#using-eksctl)
   - [Using AWS CLI](#using-aws-cli)
   - [Using Terraform](#using-terraform)
5. [Networking](#networking)
   - [VPC Design](#vpc-design)
   - [Subnets](#subnets)
   - [Security Groups](#security-groups)
   - [VPC CNI Plugin](#vpc-cni-plugin)
6. [Node Groups](#node-groups)
   - [Managed Node Groups](#managed-node-groups)
   - [Self-Managed Node Groups](#self-managed-node-groups)
   - [Fargate Profiles](#fargate-profiles)
7. [Storage](#storage)
   - [EBS CSI Driver](#ebs-csi-driver)
   - [EFS CSI Driver](#efs-csi-driver)
   - [StorageClasses](#storageclasses)
8. [Load Balancing](#load-balancing)
   - [AWS Load Balancer Controller](#aws-load-balancer-controller)
   - [ALB Ingress](#alb-ingress)
   - [NLB Services](#nlb-services)
9. [Authentication and Authorization](#authentication-and-authorization)
   - [aws-auth ConfigMap](#aws-auth-configmap)
   - [IRSA — IAM Roles for Service Accounts](#irsa-iam-roles-for-service-accounts)
   - [EKS Pod Identity](#eks-pod-identity)
10. [Cluster Autoscaler and Karpenter](#cluster-autoscaler-and-karpenter)
    - [Cluster Autoscaler](#cluster-autoscaler)
    - [Karpenter](#karpenter)
11. [Monitoring](#monitoring)
    - [CloudWatch Container Insights](#cloudwatch-container-insights)
    - [Prometheus and Grafana](#prometheus-and-grafana)
12. [Cost Optimization](#cost-optimization)
13. [Best Practices](#best-practices)
14. [Next Steps](#next-steps)

## Overview

This document covers **Amazon Elastic Kubernetes Service (EKS)** — AWS's managed Kubernetes offering that runs the Kubernetes control plane across multiple Availability Zones, automatically manages the availability and scalability of API servers and the etcd cluster, and integrates deeply with AWS services for networking, security, and observability.

### Target Audience

- DevOps and platform engineers deploying workloads on AWS
- Developers building and shipping containerized applications to EKS
- SREs operating production Kubernetes clusters on AWS
- Architects evaluating managed Kubernetes options in the cloud

### Scope

- EKS architecture, control plane, and data plane options
- Cluster creation with eksctl, AWS CLI, and Terraform
- Networking (VPC CNI), storage (EBS/EFS CSI), and load balancing (ALB/NLB)
- IAM integration, IRSA, Pod Identity, and RBAC
- Autoscaling with Cluster Autoscaler and Karpenter
- Monitoring, cost optimization, and operational best practices

## EKS Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AWS Region                                   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │              EKS Control Plane (AWS-managed)                 │   │
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
│  │  AZ-a               AZ-b               AZ-c                 │   │
│  │  ┌──────────┐      ┌──────────┐      ┌──────────┐          │   │
│  │  │  Node    │      │  Node    │      │  Fargate  │          │   │
│  │  │ ┌──┐┌──┐│      │ ┌──┐┌──┐│      │  ┌──┐     │          │   │
│  │  │ │P1││P2││      │ │P3││P4││      │  │P5│     │          │   │
│  │  │ └──┘└──┘│      │ └──┘└──┘│      │  └──┘     │          │   │
│  │  └──────────┘      └──────────┘      └──────────┘          │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Control Plane

AWS fully manages the EKS control plane. You do **not** provision, patch, or scale API servers or etcd. Key characteristics:

| Aspect | Detail |
|---|---|
| **API servers** | Distributed across at least 3 Availability Zones |
| **etcd** | Managed, encrypted, backed up automatically |
| **SLA** | 99.95 % uptime for the control plane |
| **Upgrades** | AWS provides in-place Kubernetes version upgrades |
| **Endpoint access** | Public, private, or both (configurable) |

### Data Plane

You choose how to run your workloads:

| Option | Description | Use Case |
|---|---|---|
| **Managed node groups** | AWS provisions and manages EC2 instances | General-purpose, production workloads |
| **Self-managed nodes** | You manage the EC2 Auto Scaling Groups | Custom AMIs, GPU nodes, specialized configs |
| **Fargate** | Serverless — AWS runs each Pod on a dedicated micro-VM | Burst workloads, low-ops environments |

## Prerequisites and IAM Setup

### Required Tools

Install the following CLI tools before working with EKS:

```bash
# AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# eksctl
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && sudo mv /tmp/eksctl /usr/local/bin

# Verify installations
aws --version
kubectl version --client
eksctl version
```

### IAM Roles and Policies

EKS requires two primary IAM roles:

**1. Cluster Service Role** — allows the EKS service to manage AWS resources:

```bash
# Create the cluster role trust policy
cat > eks-cluster-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role and attach the managed policy
aws iam create-role \
  --role-name EKSClusterRole \
  --assume-role-policy-document file://eks-cluster-trust-policy.json

aws iam attach-role-policy \
  --role-name EKSClusterRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```

**2. Node IAM Role** — allows EC2 instances to join the cluster:

```bash
# Create the node role trust policy
cat > eks-node-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name EKSNodeRole \
  --assume-role-policy-document file://eks-node-trust-policy.json

# Attach required policies
aws iam attach-role-policy --role-name EKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name EKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
aws iam attach-role-policy --role-name EKSNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

## Creating an EKS Cluster

### Using eksctl

`eksctl` is the official CLI tool for creating and managing EKS clusters. It abstracts away the underlying CloudFormation stacks.

**Quick start (defaults):**

```bash
eksctl create cluster \
  --name my-cluster \
  --region us-west-2 \
  --version 1.31 \
  --nodegroup-name standard-nodes \
  --node-type t3.medium \
  --nodes 3
```

**Production-ready YAML configuration:**

```yaml
# cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: production-cluster
  region: us-west-2
  version: "1.31"

iam:
  withOIDC: true  # Required for IRSA

vpc:
  cidr: 10.0.0.0/16
  nat:
    gateway: HighlyAvailable  # One NAT GW per AZ

managedNodeGroups:
  - name: system
    instanceType: m5.large
    desiredCapacity: 2
    minSize: 2
    maxSize: 4
    labels:
      role: system
    tags:
      Environment: production
    privateNetworking: true
    iam:
      withAddonPolicies:
        ebs: true
        efs: true
        albIngress: true
        cloudWatch: true

  - name: applications
    instanceType: m5.xlarge
    desiredCapacity: 3
    minSize: 1
    maxSize: 10
    labels:
      role: applications
    privateNetworking: true
    spot: false

fargateProfiles:
  - name: batch-jobs
    selectors:
      - namespace: batch
        labels:
          compute: fargate

addons:
  - name: vpc-cni
    version: latest
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: aws-ebs-csi-driver
    version: latest

cloudWatch:
  clusterLogging:
    enableTypes:
      - api
      - audit
      - authenticator
      - controllerManager
      - scheduler
```

```bash
eksctl create cluster -f cluster-config.yaml
```

### Using AWS CLI

For tighter control, you can create the cluster directly through the AWS CLI. This approach requires you to pre-create the VPC and subnets.

```bash
# Step 1 — Create the cluster
aws eks create-cluster \
  --name my-cluster \
  --region us-west-2 \
  --kubernetes-version 1.31 \
  --role-arn arn:aws:iam::123456789012:role/EKSClusterRole \
  --resources-vpc-config \
    subnetIds=subnet-0a1b2c3d,subnet-4e5f6g7h,subnet-8i9j0k1l,\
endpointPublicAccess=true,endpointPrivateAccess=true

# Step 2 — Wait for the cluster to become ACTIVE
aws eks wait cluster-active --name my-cluster --region us-west-2

# Step 3 — Update kubeconfig
aws eks update-kubeconfig --name my-cluster --region us-west-2

# Step 4 — Create a managed node group
aws eks create-nodegroup \
  --cluster-name my-cluster \
  --nodegroup-name standard-nodes \
  --node-role arn:aws:iam::123456789012:role/EKSNodeRole \
  --subnets subnet-0a1b2c3d subnet-4e5f6g7h subnet-8i9j0k1l \
  --instance-types t3.medium \
  --scaling-config minSize=2,maxSize=5,desiredSize=3

# Step 5 — Verify nodes
kubectl get nodes
```

### Using Terraform

Terraform provides a declarative, version-controlled approach to infrastructure. The example below uses the official AWS provider.

```hcl
# main.tf
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}

# ---------- VPC ----------
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.cluster_name}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["${var.region}a", "${var.region}b", "${var.region}c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = false
  enable_dns_hostnames = true

  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }
}

# ---------- EKS Cluster ----------
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = "1.31"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true

  eks_managed_node_groups = {
    system = {
      ami_type       = "AL2023_x86_64_STANDARD"
      instance_types = ["m5.large"]
      min_size       = 2
      max_size       = 4
      desired_size   = 2
    }
    applications = {
      ami_type       = "AL2023_x86_64_STANDARD"
      instance_types = ["m5.xlarge"]
      min_size       = 1
      max_size       = 10
      desired_size   = 3
    }
  }

  cluster_addons = {
    vpc-cni            = { most_recent = true }
    coredns            = { most_recent = true }
    kube-proxy         = { most_recent = true }
    aws-ebs-csi-driver = { most_recent = true }
  }
}

# ---------- Variables ----------
variable "region" {
  default = "us-west-2"
}

variable "cluster_name" {
  default = "production-cluster"
}

# ---------- Outputs ----------
output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "cluster_name" {
  value = module.eks.cluster_name
}
```

```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan
aws eks update-kubeconfig --name production-cluster --region us-west-2
```

## Networking

### VPC Design

EKS clusters run inside a VPC. A well-designed VPC uses multiple Availability Zones with separate public and private subnets.

```
┌─────────────────────────── VPC 10.0.0.0/16 ───────────────────────────┐
│                                                                        │
│  AZ-a                    AZ-b                    AZ-c                  │
│  ┌────────────────┐     ┌────────────────┐     ┌────────────────┐     │
│  │ Public Subnet  │     │ Public Subnet  │     │ Public Subnet  │     │
│  │ 10.0.101.0/24  │     │ 10.0.102.0/24  │     │ 10.0.103.0/24  │     │
│  │  ┌──────────┐  │     │  ┌──────────┐  │     │  ┌──────────┐  │     │
│  │  │ NAT GW   │  │     │  │ NAT GW   │  │     │  │ NAT GW   │  │     │
│  │  └────┬─────┘  │     │  └────┬─────┘  │     │  └────┬─────┘  │     │
│  └───────┼────────┘     └───────┼────────┘     └───────┼────────┘     │
│          │                      │                      │               │
│  ┌───────▼────────┐     ┌───────▼────────┐     ┌───────▼────────┐     │
│  │ Private Subnet │     │ Private Subnet │     │ Private Subnet │     │
│  │ 10.0.1.0/24    │     │ 10.0.2.0/24    │     │ 10.0.3.0/24    │     │
│  │ ┌────┐ ┌────┐  │     │ ┌────┐ ┌────┐  │     │ ┌────┐ ┌────┐  │     │
│  │ │Node│ │Node│  │     │ │Node│ │Node│  │     │ │Node│ │Node│  │     │
│  │ └────┘ └────┘  │     │ └────┘ └────┘  │     │ └────┘ └────┘  │     │
│  └────────────────┘     └────────────────┘     └────────────────┘     │
│                                                                        │
│  ┌──────────┐                                                          │
│  │ IGW      │  ← Internet Gateway (attached to VPC)                    │
│  └──────────┘                                                          │
└────────────────────────────────────────────────────────────────────────┘
```

### Subnets

| Subnet Type | Purpose | Routing | Required Tags |
|---|---|---|---|
| **Public** | Load balancers (ALB/NLB), bastion hosts | → Internet Gateway | `kubernetes.io/role/elb = 1` |
| **Private** | Worker nodes, Pods | → NAT Gateway | `kubernetes.io/role/internal-elb = 1` |

> **Tip:** Always place worker nodes in private subnets. Expose services through load balancers in public subnets.

### Security Groups

EKS creates a **cluster security group** automatically. Key rules:

| Direction | Port | Source / Destination | Purpose |
|---|---|---|---|
| Inbound | 443 | Worker nodes SG | kubelet → API server |
| Inbound | 443 | Admin CIDR | kubectl access |
| Outbound | 10250 | Worker nodes SG | API server → kubelet |
| Outbound | 443 | 0.0.0.0/0 | AWS API calls |

### VPC CNI Plugin

The **Amazon VPC CNI plugin** (`aws-node`) assigns real VPC IP addresses to each Pod. This means every Pod is directly routable within the VPC without overlays.

```
┌────────── EC2 Instance (m5.large) ──────────┐
│  Primary ENI (eth0)                          │
│    Primary IP: 10.0.1.10                     │
│    Secondary IPs: 10.0.1.11, 10.0.1.12, ... │
│                                              │
│  ┌──────┐  ┌──────┐  ┌──────┐               │
│  │ Pod  │  │ Pod  │  │ Pod  │               │
│  │.1.11 │  │.1.12 │  │.1.13 │               │
│  └──────┘  └──────┘  └──────┘               │
│                                              │
│  Secondary ENI (eth1)  ← added as needed     │
│    Secondary IPs: 10.0.1.20, 10.0.1.21, ... │
└──────────────────────────────────────────────┘
```

Key configuration environment variables for the `aws-node` DaemonSet:

| Variable | Default | Description |
|---|---|---|
| `WARM_ENI_TARGET` | `1` | Number of spare ENIs to keep attached |
| `MINIMUM_IP_TARGET` | — | Minimum free IPs to maintain |
| `ENABLE_PREFIX_DELEGATION` | `false` | Assign /28 prefixes instead of individual IPs (increases Pod density) |

```bash
# Enable prefix delegation for higher Pod density
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true
kubectl set env daemonset aws-node -n kube-system WARM_PREFIX_TARGET=1
```

## Node Groups

### Managed Node Groups

AWS provisions and manages the underlying EC2 instances, including AMI updates, draining, and replacement.

```bash
# Create via eksctl
eksctl create nodegroup \
  --cluster my-cluster \
  --name app-nodes \
  --node-type m5.xlarge \
  --nodes-min 2 \
  --nodes-max 10 \
  --managed

# List node groups
eksctl get nodegroup --cluster my-cluster

# Update node group
eksctl upgrade nodegroup \
  --cluster my-cluster \
  --name app-nodes \
  --kubernetes-version 1.31
```

### Self-Managed Node Groups

You control the launch template, AMI, and Auto Scaling Group. Useful when you need custom AMIs or specialized instance types (GPU, Arm).

```bash
# Retrieve the EKS-optimized AMI ID
aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.31/amazon-linux-2023/x86_64/standard/recommended/image_id \
  --query "Parameter.Value" --output text
```

### Fargate Profiles

Fargate runs each Pod in its own isolated micro-VM. No nodes to manage — you only define which Pods should run on Fargate.

```yaml
# fargate-profile.yaml (eksctl)
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-west-2

fargateProfiles:
  - name: default-fargate
    selectors:
      - namespace: default
        labels:
          compute: fargate
      - namespace: kube-system
        labels:
          compute: fargate
```

**Fargate limitations:**

- No DaemonSets (each Pod gets its own micro-VM)
- No GPU instances
- No persistent volumes backed by EBS (EFS is supported)
- Maximum 20 GB ephemeral storage per Pod

## Storage

### EBS CSI Driver

The **Amazon EBS CSI driver** provisions block storage volumes for Pods. It is installed as an EKS add-on.

```bash
# Install as an EKS add-on
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name aws-ebs-csi-driver \
  --service-account-role-arn arn:aws:iam::123456789012:role/EBS_CSI_DriverRole
```

### EFS CSI Driver

The **Amazon EFS CSI driver** provides shared file storage accessible by multiple Pods across Availability Zones.

```bash
# Install via Helm
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::123456789012:role/EFS_CSI_DriverRole
```

### StorageClasses

```yaml
# gp3-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
  iops: "3000"
  throughput: "125"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
---
# efs-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-0123456789abcdef0
  directoryPerms: "700"
```

```bash
kubectl apply -f gp3-storageclass.yaml
kubectl apply -f efs-storageclass.yaml
```

## Load Balancing

### AWS Load Balancer Controller

The **AWS Load Balancer Controller** provisions ALBs and NLBs in response to Kubernetes Ingress and Service resources. It replaces the legacy in-tree cloud provider load balancer.

```bash
# Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=true \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::123456789012:role/AmazonEKSLoadBalancerControllerRole
```

### ALB Ingress

An **Application Load Balancer** (Layer 7) is provisioned when you create an Ingress with the `alb` class.

```yaml
# alb-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-west-2:123456789012:certificate/abc-123
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
spec:
  ingressClassName: alb
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

### NLB Services

A **Network Load Balancer** (Layer 4) is created for Services of type `LoadBalancer`.

```yaml
# nlb-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: app-nlb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 443
      targetPort: 8443
      protocol: TCP
```

```
                 Internet
                    │
           ┌───────▼───────┐
           │  ALB or NLB   │     ← provisioned by LB Controller
           └───┬───┬───┬───┘
               │   │   │
          ┌────▼┐┌─▼──┐┌▼────┐
          │ Pod ││ Pod ││ Pod │   ← target-type: ip (direct Pod IPs)
          └─────┘└────┘└─────┘
```

## Authentication and Authorization

### aws-auth ConfigMap

The `aws-auth` ConfigMap in the `kube-system` namespace maps IAM principals to Kubernetes RBAC identities. EKS uses this to authorize IAM users and roles.

```yaml
# aws-auth ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/EKSNodeRole
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::123456789012:role/AdminRole
      username: admin
      groups:
        - system:masters
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/developer
      username: developer
      groups:
        - dev-team
```

```bash
# Edit the ConfigMap (use eksctl for safety)
eksctl create iamidentitymapping \
  --cluster my-cluster \
  --arn arn:aws:iam::123456789012:role/DevRole \
  --group dev-team \
  --username developer
```

### IRSA — IAM Roles for Service Accounts

**IAM Roles for Service Accounts (IRSA)** lets Pods assume fine-grained IAM roles without embedding credentials. It uses an OIDC provider to federate Kubernetes service accounts with IAM.

```
┌────────────────────┐         ┌─────────────────┐
│  Pod               │         │  AWS IAM         │
│  ┌──────────────┐  │  OIDC   │  ┌────────────┐ │
│  │ Service      │──┼────────►│  │ IAM Role   │ │
│  │ Account      │  │  trust  │  │ (scoped    │ │
│  │ (annotated)  │  │         │  │  policy)   │ │
│  └──────────────┘  │         │  └─────┬──────┘ │
└────────────────────┘         │        │        │
                               │  ┌─────▼──────┐ │
                               │  │ S3, DynamoDB│ │
                               │  │ SQS, etc.  │ │
                               │  └────────────┘ │
                               └─────────────────┘
```

```bash
# Step 1 — Associate an OIDC provider with the cluster
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve

# Step 2 — Create a service account with an IAM role
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace default \
  --name s3-reader \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

```yaml
# The resulting service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/eksctl-my-cluster-s3-reader
```

### EKS Pod Identity

**EKS Pod Identity** (GA since 2024) is the recommended successor to IRSA. It simplifies IAM-to-Pod mapping by eliminating the need for OIDC provider management.

```bash
# Install the Pod Identity Agent add-on
aws eks create-addon --cluster-name my-cluster --addon-name eks-pod-identity-agent

# Create a Pod Identity association
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace default \
  --service-account s3-reader \
  --role-arn arn:aws:iam::123456789012:role/S3ReaderRole
```

| Feature | IRSA | Pod Identity |
|---|---|---|
| OIDC provider required | Yes | No |
| Cross-account support | Manual trust policy | Built-in |
| Setup complexity | Medium | Low |
| Availability | All EKS versions | EKS 1.24+ |

## Cluster Autoscaler and Karpenter

### Cluster Autoscaler

The **Kubernetes Cluster Autoscaler** adjusts the size of node groups based on pending Pod resource requests.

```bash
# Install via Helm
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=us-west-2 \
  --set rbac.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::123456789012:role/ClusterAutoscalerRole
```

### Karpenter

**Karpenter** is an open-source, high-performance node provisioner built for Kubernetes on AWS. It provisions right-sized EC2 instances directly (bypasses ASGs) and responds to scheduling needs in seconds.

```yaml
# karpenter-nodepool.yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand", "spot"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["4"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: "1000"
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiSelectorTerms:
    - alias: al2023@latest
  subnetSelectorTerms:
    - tags:
        kubernetes.io/role/internal-elb: "1"
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  role: KarpenterNodeRole
```

| Feature | Cluster Autoscaler | Karpenter |
|---|---|---|
| Provisioning model | Scales existing ASGs | Launches instances directly |
| Speed | Minutes | Seconds |
| Instance diversity | Fixed per node group | Flexible, multi-instance |
| Spot support | Via ASG mixed instances | Native, first-class |
| Consolidation | Limited | Built-in, automatic |

## Monitoring

### CloudWatch Container Insights

Container Insights collects metrics and logs from EKS clusters using the CloudWatch agent and Fluent Bit.

```bash
# Enable via eksctl
eksctl utils update-cluster-logging \
  --cluster my-cluster \
  --enable-types all \
  --approve

# Install the CloudWatch Observability add-on
aws eks create-addon \
  --cluster-name my-cluster \
  --addon-name amazon-cloudwatch-observability
```

### Prometheus and Grafana

For open-source observability, deploy **Amazon Managed Prometheus (AMP)** and **Amazon Managed Grafana (AMG)** or self-hosted equivalents.

```bash
# Install Prometheus via Helm
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

## Cost Optimization

| Strategy | Implementation | Estimated Savings |
|---|---|---|
| **Spot Instances** | Use Karpenter with `capacity-type: spot` for fault-tolerant workloads | 60–90 % vs On-Demand |
| **Right-sizing** | Deploy VPA (Vertical Pod Autoscaler) to recommend resource requests | 20–40 % |
| **Graviton instances** | Use `c7g`/`m7g` Arm instances with multi-arch images | ~20 % vs x86 |
| **Fargate for burst** | Use Fargate profiles for short-lived batch jobs | Pay-per-Pod (no idle) |
| **Savings Plans** | Commit to 1-yr or 3-yr Compute Savings Plans for baseline nodes | 30–50 % |
| **Scale to zero** | Use Karpenter consolidation to remove idle nodes | Eliminate waste |
| **Namespace quotas** | Set `ResourceQuota` and `LimitRange` per namespace | Prevent sprawl |

```bash
# Check current node utilization
kubectl top nodes

# Find Pods without resource requests (cost risk)
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.containers[].resources.requests == null) |
  "\(.metadata.namespace)/\(.metadata.name)"'
```

## Best Practices

### Security

- **Enable envelope encryption** for Kubernetes secrets using an AWS KMS key
- **Use private endpoint access** and restrict public access to known CIDRs
- **Use IRSA or Pod Identity** — never use node-level IAM roles for application workloads
- **Enable audit logging** to CloudWatch and monitor with GuardDuty for EKS
- **Scan container images** with Amazon ECR image scanning or a third-party tool
- **Apply Pod Security Standards** using Kubernetes admission controllers

### Reliability

- **Spread across 3+ AZs** using pod topology spread constraints
- **Set resource requests and limits** on all containers to avoid OOM kills and noisy neighbors
- **Use Pod Disruption Budgets (PDBs)** for critical workloads during upgrades
- **Run CoreDNS with anti-affinity** across nodes for DNS resilience
- **Use managed node groups** for simplified patching and automated drain-and-replace upgrades

### Operations

- **Automate upgrades** — keep control plane and node groups within one minor version
- **Tag all resources** with `Environment`, `Team`, and `CostCenter` for cost allocation
- **Use GitOps** (Flux or Argo CD) for declarative cluster configuration
- **Store Terraform state remotely** in S3 with DynamoDB locking
- **Test upgrades in a staging cluster** before rolling out to production

### Networking

- **Enable prefix delegation** on the VPC CNI if you need higher Pod density
- **Use Security Groups for Pods** to apply per-Pod network policies via AWS SGs
- **Deploy the AWS Load Balancer Controller** instead of the in-tree cloud provider
- **Reserve IP space** — ensure your VPC CIDR has room for future growth

## Next Steps

- [Kubernetes Services](01-SERVICES.md) — revisit service types and traffic routing
- **Helm and Package Management** — managing application charts on EKS
- **CI/CD Pipelines** — deploy to EKS from CodePipeline, GitHub Actions, or Argo CD
- **Service Mesh** — Istio or AWS App Mesh for advanced traffic management
- **Multi-Cluster** — federation, failover, and multi-region EKS patterns
