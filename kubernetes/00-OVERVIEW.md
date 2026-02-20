# Kubernetes Overview

## Table of Contents

1. [Overview](#overview)
2. [What is Kubernetes](#what-is-kubernetes)
3. [Kubernetes Architecture](#kubernetes-architecture)
4. [Core Concepts](#core-concepts)
5. [Kubernetes API and kubectl](#kubernetes-api-and-kubectl)
6. [Key Components Diagram](#key-components-diagram)
7. [Common Kubernetes Distributions](#common-kubernetes-distributions)
8. [Version History and Release Cycle](#version-history-and-release-cycle)
9. [Prerequisites](#prerequisites)
10. [Next Steps](#next-steps)

## Overview

This documentation provides a comprehensive introduction to Kubernetes — the industry-standard platform for container orchestration. It covers architecture, core concepts, tooling, and practical examples to get you productive with Kubernetes quickly.

### Target Audience

- Developers deploying containerized applications
- DevOps and platform engineers building internal platforms
- SREs responsible for cluster operations and reliability
- Architects evaluating container orchestration strategies

### Scope

- Kubernetes architecture and component roles
- Core resource types (Pods, Nodes, Namespaces, Services)
- Cluster interaction via the Kubernetes API and `kubectl`
- Distribution landscape (upstream, managed, enterprise)
- Release cadence and version support policy

## What is Kubernetes

Kubernetes (often abbreviated **K8s**) is an open-source container orchestration platform originally designed by Google and now maintained by the Cloud Native Computing Foundation (CNCF). It automates the deployment, scaling, and management of containerized applications across clusters of machines.

### Key Capabilities

| Capability | Description |
|---|---|
| **Service discovery & load balancing** | Exposes containers via DNS or IP and distributes traffic |
| **Storage orchestration** | Mounts local, cloud, or network storage automatically |
| **Automated rollouts & rollbacks** | Declaratively change the desired state; K8s handles the transition |
| **Self-healing** | Restarts failed containers, replaces nodes, kills unresponsive pods |
| **Secret & configuration management** | Stores and manages sensitive data separately from images |
| **Horizontal scaling** | Scales workloads up or down based on CPU, memory, or custom metrics |
| **Batch execution** | Manages batch and CI workloads alongside long-running services |

### Why Kubernetes?

```
Traditional Deployment        Virtualized Deployment        Container Orchestration
┌──────────────────┐         ┌──────────────────┐         ┌──────────────────┐
│   App A   App B  │         │  ┌─────┐ ┌─────┐ │         │ ┌──┐ ┌──┐ ┌──┐  │
│                  │         │  │VM A │ │VM B │ │         │ │C1│ │C2│ │C3│  │
│   Runtime Deps   │         │  │App A│ │App B│ │         │ └──┘ └──┘ └──┘  │
│                  │         │  │ Libs │ │ Libs│ │         │   Container      │
│                  │         │  │  OS  │ │  OS │ │         │    Runtime       │
│   Host OS        │         │  └─────┘ └─────┘ │         │                  │
│   Hardware       │         │  Hypervisor       │         │   Host OS        │
└──────────────────┘         │  Host OS          │         │   Hardware       │
                             └──────────────────┘         └──────────────────┘
  No isolation                 Heavy, slow scaling           Lightweight, fast
```

## Kubernetes Architecture

A Kubernetes cluster is divided into two logical layers: the **control plane** and the **data plane** (worker nodes).

### Control Plane

The control plane manages the overall state of the cluster. It makes scheduling decisions, detects and responds to cluster events, and exposes the Kubernetes API.

#### kube-apiserver

The front door to the cluster. Every interaction — from `kubectl` commands to internal component communication — goes through the API server.

```bash
# The API server validates and processes all RESTful requests
# Example: query the API server directly
curl -k https://<control-plane-ip>:6443/api/v1/namespaces/default/pods \
  -H "Authorization: Bearer $TOKEN"
```

#### etcd

A distributed, consistent key-value store used as the backing store for all cluster data. Every resource definition, configuration, and state record lives in etcd.

- Stores the desired and actual state of every resource
- Supports watch operations for real-time change notification
- Should be backed up regularly in production

#### kube-scheduler

Watches for newly created Pods with no assigned node, and selects a node for them to run on based on resource requirements, constraints, affinity rules, and data locality.

#### kube-controller-manager

Runs a collection of control loops (controllers) that regulate the state of the cluster:

| Controller | Responsibility |
|---|---|
| Node Controller | Detects and responds when nodes go down |
| ReplicaSet Controller | Maintains the correct number of Pods |
| Endpoints Controller | Populates the Endpoints object for Services |
| ServiceAccount Controller | Creates default ServiceAccounts for new Namespaces |

### Data Plane (Worker Nodes)

Worker nodes run the actual application workloads. Each node runs the following components:

#### kubelet

The primary agent on each node. It ensures containers described in PodSpecs are running and healthy.

```yaml
# kubelet configuration snippet (KubeletConfiguration)
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
maxPods: 110
containerRuntimeEndpoint: unix:///var/run/containerd/containerd.sock
```

#### kube-proxy

Maintains network rules on each node, implementing Kubernetes Service abstraction by forwarding traffic to the appropriate Pod backends. It can operate in iptables or IPVS mode.

#### Container Runtime

The software responsible for running containers. Kubernetes supports any runtime that implements the **Container Runtime Interface (CRI)**:

- **containerd** — industry default, lightweight, CNCF graduated
- **CRI-O** — optimized for Kubernetes, used by OpenShift
- **Docker Engine** — supported via the cri-dockerd adapter (dockershim removed in v1.24)

## Core Concepts

### Pods

A Pod is the smallest deployable unit in Kubernetes. It represents one or more containers that share networking and storage.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    environment: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "250m"
          memory: "256Mi"
```

```bash
# Create the Pod
kubectl apply -f nginx-pod.yaml

# Check Pod status
kubectl get pods -o wide

# View Pod logs
kubectl logs nginx-pod

# Execute a command inside the Pod
kubectl exec -it nginx-pod -- /bin/bash
```

### Nodes

A Node is a worker machine (physical or virtual) in the Kubernetes cluster. Each Node is managed by the control plane and runs kubelet, kube-proxy, and a container runtime.

```bash
# List all nodes and their status
kubectl get nodes

# Describe a specific node
kubectl describe node <node-name>

# Cordon a node (prevent new Pods from being scheduled)
kubectl cordon <node-name>

# Drain a node (evict all Pods for maintenance)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

### Namespaces

Namespaces provide logical isolation within a cluster. They scope resource names and can be used to apply resource quotas and access policies.

```bash
# List all namespaces
kubectl get namespaces

# Create a namespace
kubectl create namespace staging

# Set default namespace for current context
kubectl config set-context --current --namespace=staging
```

Default namespaces in every cluster:

| Namespace | Purpose |
|---|---|
| `default` | Resources with no explicit namespace |
| `kube-system` | Control plane and system components |
| `kube-public` | Publicly readable resources (e.g., cluster-info) |
| `kube-node-lease` | Node heartbeat Lease objects for health detection |

### Labels and Selectors

Labels are key-value pairs attached to objects. Selectors query objects by their labels.

```yaml
# Attaching labels to a Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web
    tier: frontend
    release: stable
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        tier: frontend
    spec:
      containers:
        - name: web
          image: my-web-app:2.1.0
```

```bash
# Filter resources by label
kubectl get pods -l app=web

# Multiple label selectors
kubectl get pods -l "app=web,tier=frontend"

# Set-based selectors
kubectl get pods -l "tier in (frontend, backend)"

# Add a label to an existing resource
kubectl label pod nginx-pod version=v1
```

## Kubernetes API and kubectl

### API Structure

The Kubernetes API is a RESTful interface organized into API groups and versions:

```
/api/v1                        # Core (legacy) API group
/apis/apps/v1                  # Apps API group (Deployments, StatefulSets)
/apis/batch/v1                 # Batch API group (Jobs, CronJobs)
/apis/networking.k8s.io/v1     # Networking API group (Ingress, NetworkPolicy)
/apis/rbac.authorization.k8s.io/v1  # RBAC API group
```

### Essential kubectl Commands

```bash
# Cluster information
kubectl cluster-info
kubectl version --client
kubectl api-resources          # List all available resource types

# Working with resources
kubectl get <resource>              # List resources
kubectl describe <resource> <name>  # Detailed info
kubectl create -f <file.yaml>       # Create from manifest
kubectl apply -f <file.yaml>        # Create or update declaratively
kubectl delete <resource> <name>    # Remove a resource

# Debugging
kubectl logs <pod-name> -f                   # Stream logs
kubectl logs <pod-name> -c <container-name>  # Multi-container Pod
kubectl exec -it <pod-name> -- sh            # Interactive shell
kubectl port-forward <pod-name> 8080:80      # Local port forwarding
kubectl top pods                             # Resource usage (requires metrics-server)

# Context and configuration
kubectl config get-contexts           # List available contexts
kubectl config use-context <name>     # Switch cluster context
kubectl config view --minify          # Show current context details
```

### Declarative vs. Imperative

```bash
# Imperative — quick but not reproducible
kubectl create deployment nginx --image=nginx:1.27 --replicas=3

# Declarative — preferred for production
kubectl apply -f deployment.yaml
```

❌ **Don't:**
- Use imperative commands for production workloads
- Store manifests without version control
- Run `kubectl` against production without verifying the current context

✅ **Do:**
- Store all manifests in Git (GitOps)
- Use `kubectl diff -f manifest.yaml` before applying changes
- Set up RBAC to restrict access per namespace
- Use `kubectl apply` with `--dry-run=client -o yaml` to preview changes

## Key Components Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CONTROL PLANE                              │
│                                                                     │
│  ┌────────────┐  ┌─────────────────┐  ┌──────────────────────────┐ │
│  │  kube-api   │  │  kube-scheduler │  │ kube-controller-manager  │ │
│  │   server    │  │                 │  │                          │ │
│  │             │  │  Assigns Pods   │  │  Node / ReplicaSet /     │ │
│  │ REST API    │◄─┤  to Nodes       │  │  Endpoints / SA          │ │
│  │ AuthN/AuthZ │  │                 │  │  Controllers             │ │
│  └──────┬──────┘  └─────────────────┘  └──────────────────────────┘ │
│         │                                                           │
│         ▼                                                           │
│  ┌────────────┐         ┌──────────────────────────────────┐       │
│  │   etcd     │         │  cloud-controller-manager        │       │
│  │            │         │  (optional — cloud providers)    │       │
│  │ Key-Value  │         └──────────────────────────────────┘       │
│  │ Store      │                                                     │
│  └────────────┘                                                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               │  Kubernetes API
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         DATA PLANE (Worker Nodes)                   │
│                                                                      │
│  ┌─────────────────────────────┐  ┌──────────────────────────────┐  │
│  │         Node 1              │  │         Node 2               │  │
│  │                             │  │                              │  │
│  │  ┌────────┐  ┌───────────┐ │  │  ┌────────┐  ┌────────────┐ │  │
│  │  │kubelet │  │kube-proxy │ │  │  │kubelet │  │kube-proxy  │ │  │
│  │  └────────┘  └───────────┘ │  │  └────────┘  └────────────┘ │  │
│  │                             │  │                              │  │
│  │  ┌─────────────────────┐   │  │  ┌──────────────────────┐   │  │
│  │  │  Container Runtime  │   │  │  │  Container Runtime   │   │  │
│  │  │  (containerd)       │   │  │  │  (containerd)        │   │  │
│  │  └─────────────────────┘   │  │  └──────────────────────┘   │  │
│  │                             │  │                              │  │
│  │  ┌──────┐ ┌──────┐ ┌────┐ │  │  ┌──────┐ ┌──────┐         │  │
│  │  │Pod A │ │Pod B │ │PodC│ │  │  │Pod D │ │Pod E │         │  │
│  │  └──────┘ └──────┘ └────┘ │  │  └──────┘ └──────┘         │  │
│  └─────────────────────────────┘  └──────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

## Common Kubernetes Distributions

### Distribution Comparison

| Distribution | Provider | Environment | Key Differentiator |
|---|---|---|---|
| **Upstream (kubeadm)** | CNCF | Self-managed | Vanilla Kubernetes; full control |
| **Amazon EKS** | AWS | Managed | Deep AWS integration (IAM, VPC, ALB) |
| **Azure AKS** | Microsoft | Managed | Free control plane; Azure AD integration |
| **Google GKE** | Google | Managed | Autopilot mode; fastest upstream adoption |
| **Oracle OKE** | Oracle | Managed | OCI integration; bare-metal node support |
| **Red Hat OpenShift** | Red Hat | Enterprise | Built-in CI/CD, registry, and developer console |
| **k3s** | SUSE/Rancher | Edge / IoT | Lightweight (~70 MB binary); ARM support |
| **kind** | CNCF SIG | Development | Runs clusters inside Docker containers |
| **minikube** | CNCF SIG | Development | Local single-node cluster for learning |

### Choosing a Distribution

```
Is this production?
├── Yes
│   ├── Running in a public cloud? → Use the cloud provider's managed offering
│   │                                 (EKS, AKS, GKE, OKE)
│   └── On-premises or multi-cloud? → OpenShift or upstream with kubeadm
└── No
    ├── Local development?          → minikube, kind, or k3s
    └── CI/CD pipelines?            → kind (fast startup, disposable)
```

## Version History and Release Cycle

Kubernetes follows a **three releases per year** cadence. Each minor release is supported with patch updates for approximately **14 months** (12 months standard + 2 months extended).

### Release Support Policy

```
Release     GA Date         End of Life       Status
─────────────────────────────────────────────────────
v1.32       Dec 2024        Feb 2026          Current
v1.31       Aug 2024        Oct 2025          Supported
v1.30       Apr 2024        Jun 2025          Supported
v1.29       Dec 2023        Feb 2025          End of Life
v1.28       Aug 2023        Oct 2024          End of Life
```

### Notable Version Milestones

| Version | Milestone |
|---|---|
| v1.0 (2015) | First stable release; CNCF donation |
| v1.7 (2017) | RBAC promoted to beta |
| v1.16 (2019) | CRDs reach GA; legacy API versions removed |
| v1.20 (2020) | Dockershim deprecation announced |
| v1.24 (2022) | Dockershim removed; containerd/CRI-O required |
| v1.25 (2022) | PodSecurityPolicy removed; Pod Security Admission GA |
| v1.27 (2023) | In-place Pod resource resize (alpha) |
| v1.30 (2024) | Structured authorization configuration GA |

### Upgrade Best Practices

- Upgrade one minor version at a time (e.g., 1.30 → 1.31 → 1.32)
- Read the changelog and migration guide for each release
- Test upgrades in a staging environment before production
- Back up etcd before every control plane upgrade

## Prerequisites

Before working through the documents in this series, ensure you have the following:

### Required Knowledge

- **Linux fundamentals** — filesystem navigation, process management, networking basics
- **Containers** — building images with Dockerfiles, running containers, image registries
- **YAML syntax** — Kubernetes manifests are defined in YAML
- **Networking basics** — TCP/IP, DNS, HTTP, ports, and load balancing concepts

### Required Tools

```bash
# kubectl — Kubernetes CLI
# Install on Linux
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Verify installation
kubectl version --client

# minikube — local Kubernetes cluster for learning
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube-linux-amd64 && sudo mv minikube-linux-amd64 /usr/local/bin/minikube

# Start a local cluster
minikube start

# helm — Kubernetes package manager
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Recommended Tools

| Tool | Purpose |
|---|---|
| `k9s` | Terminal-based UI for cluster interaction |
| `kubectx` / `kubens` | Fast context and namespace switching |
| `stern` | Multi-pod log tailing |
| `kustomize` | Template-free manifest customization (built into kubectl) |
| `lens` | Desktop GUI for Kubernetes cluster management |

## Document Structure

```
kubernetes/
├── 00-OVERVIEW.md              (this file)
├── 01-SERVICES.md
├── 02-CLOUD-SERVICES-AWS-EKS.md
├── 03-CLOUD-SERVICES-OKE.md
├── 04-CLOUD-SERVICES-AKS.md
├── 05-LOCAL-SETUP-KIND-MINIKUBE.md
├── 06-MANIFESTS.md
├── 07-CRDS-AND-OPERATORS.md
├── 08-CONTROLLERS.md
├── 09-NETWORKING.md
├── 10-BEST-PRACTICES-AND-PATTERNS.md
├── 11-ANTI-PATTERNS.md
├── LEARNING-PATH.md
└── README.md
```

## Next Steps

Continue to [Services](01-SERVICES.md) to learn about ClusterIP, NodePort, LoadBalancer, Ingress, and service mesh patterns.

### Suggested Learning Path

1. **[Services](01-SERVICES.md)** — ClusterIP, NodePort, LoadBalancer, Ingress, service mesh
2. **[AWS EKS](02-CLOUD-SERVICES-AWS-EKS.md)** — Managed Kubernetes on Amazon Web Services
3. **[Oracle OKE](03-CLOUD-SERVICES-OKE.md)** — Managed Kubernetes on Oracle Cloud
4. **[Azure AKS](04-CLOUD-SERVICES-AKS.md)** — Managed Kubernetes on Microsoft Azure
5. **[Local Setup (kind & Minikube)](05-LOCAL-SETUP-KIND-MINIKUBE.md)** — Local development clusters
6. **[Manifests](06-MANIFESTS.md)** — Kubernetes YAML manifests and resource definitions
7. **[CRDs & Operators](07-CRDS-AND-OPERATORS.md)** — Custom Resource Definitions and Operator pattern
8. **[Controllers](08-CONTROLLERS.md)** — Custom controllers and reconciliation loops
9. **[Networking](09-NETWORKING.md)** — CNI plugins, network policies, and DNS
10. **[Best Practices & Patterns](10-BEST-PRACTICES-AND-PATTERNS.md)** — Production-ready guidelines
11. **[Anti-Patterns](11-ANTI-PATTERNS.md)** — Common mistakes to avoid

## Support and Resources

### Official Documentation

- **Kubernetes Docs**: https://kubernetes.io/docs/
- **Kubernetes API Reference**: https://kubernetes.io/docs/reference/kubernetes-api/
- **Kubernetes Blog**: https://kubernetes.io/blog/

### Community

- **CNCF Slack**: https://slack.cncf.io/ — `#kubernetes-users` channel
- **Kubernetes GitHub**: https://github.com/kubernetes/kubernetes
- **KubeCon + CloudNativeCon**: Annual CNCF conferences

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Kubernetes overview documentation |
