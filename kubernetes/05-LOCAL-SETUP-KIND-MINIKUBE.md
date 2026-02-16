# Local Kubernetes Setup — kind and Minikube

## Table of Contents

1. [Overview](#overview)
2. [Why Local Kubernetes](#why-local-kubernetes)
3. [kind — Kubernetes IN Docker](#kind--kubernetes-in-docker)
   - [Installing kind](#installing-kind)
   - [Creating Clusters](#creating-clusters)
   - [Cluster Configuration](#cluster-configuration)
   - [Loading Local Images](#loading-local-images)
   - [Ingress Setup with kind](#ingress-setup-with-kind)
   - [Persistent Storage](#persistent-storage)
   - [Common kind Commands](#common-kind-commands)
4. [Minikube](#minikube)
   - [Installing Minikube](#installing-minikube)
   - [Drivers](#drivers)
   - [Creating Clusters and Profiles](#creating-clusters-and-profiles)
   - [Minikube Addons](#minikube-addons)
   - [Mounting Host Directories](#mounting-host-directories)
   - [LoadBalancer Services with minikube tunnel](#loadbalancer-services-with-minikube-tunnel)
   - [Multi-Node Clusters](#multi-node-clusters)
   - [Common Minikube Commands](#common-minikube-commands)
5. [Other Local Options](#other-local-options)
6. [Comparison Table](#comparison-table)
7. [Development Workflow with Local Clusters](#development-workflow-with-local-clusters)
   - [Building and Testing Container Images Locally](#building-and-testing-container-images-locally)
   - [Hot-Reload and Live Development](#hot-reload-and-live-development)
8. [Troubleshooting Common Issues](#troubleshooting-common-issues)
9. [Best Practices](#best-practices)
10. [Next Steps](#next-steps)

## Overview

This document covers setting up local Kubernetes clusters for development and testing using **kind** and **Minikube** — the two most widely adopted tools for running Kubernetes on a developer workstation. It includes installation, configuration, advanced usage patterns, and practical workflows for building and testing applications locally before deploying to production clusters.

### Target Audience

- Developers building and testing containerized applications locally
- DevOps engineers prototyping cluster configurations before applying to production
- SREs validating manifests, Helm charts, and operators in a safe environment
- Teams evaluating local Kubernetes tooling for their development workflow

### Scope

- Installation and configuration of kind and Minikube on Linux, macOS, and Windows
- Single-node and multi-node cluster creation
- Image loading, ingress, storage, and networking in local clusters
- Development workflow tooling (Skaffold, Tilt, Telepresence)
- Comparison of local Kubernetes options and best practices

## Why Local Kubernetes

Running Kubernetes locally allows developers to iterate quickly without relying on shared remote clusters or incurring cloud costs. Local clusters provide a production-like environment for testing manifests, debugging services, and validating application behavior before pushing changes.

### When to Use a Local Cluster

| Use Case | Benefit |
|---|---|
| **Application development** | Fast inner-loop iteration with local images |
| **Manifest validation** | Catch YAML errors before applying to staging or production |
| **CI pipelines** | Ephemeral clusters in CI for integration testing |
| **Learning and experimentation** | Safe environment to explore Kubernetes features |
| **Offline development** | No cloud or network dependency |

### Comparison of Approaches

```
Remote Shared Cluster                 Local Cluster
┌─────────────────────────┐          ┌─────────────────────────┐
│  Shared among team      │          │  Isolated per developer │
│  Network latency        │          │  No network dependency  │
│  Cloud costs            │          │  Free                   │
│  Namespace isolation     │          │  Full cluster access    │
│  Production-like infra  │          │  Lightweight            │
└─────────────────────────┘          └─────────────────────────┘
```

## kind — Kubernetes IN Docker

**kind** (Kubernetes IN Docker) runs Kubernetes cluster nodes as Docker containers. It was originally designed for testing Kubernetes itself and is now widely used for local development, CI pipelines, and integration testing. Each cluster node is a Docker container running `systemd` and `containerd`.

```
┌──────────────────────────────────────┐
│           Docker Host                │
│  ┌────────────────┐ ┌────────────┐  │
│  │ Control Plane  │ │  Worker    │  │
│  │  (container)   │ │ (container)│  │
│  │  ┌──────────┐  │ │ ┌────────┐│  │
│  │  │ kubelet  │  │ │ │kubelet ││  │
│  │  │ API srv  │  │ │ │        ││  │
│  │  │ etcd     │  │ │ │workload││  │
│  │  └──────────┘  │ │ └────────┘│  │
│  └────────────────┘ └────────────┘  │
└──────────────────────────────────────┘
```

### Installing kind

kind requires Docker (or Podman) to be installed and running.

**Linux:**

```bash
# Download the latest kind binary
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Verify installation
kind version
```

**macOS:**

```bash
# Using Homebrew
brew install kind

# Or download directly
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-darwin-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

**Windows (PowerShell):**

```powershell
# Using Chocolatey
choco install kind

# Or download directly
curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/latest/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe C:\Windows\kind.exe
```

### Creating Clusters

**Single-node cluster (default):**

```bash
# Create a cluster with the default name "kind"
kind create cluster

# Create a named cluster
kind create cluster --name my-app

# Create a cluster with a specific Kubernetes version
kind create cluster --image kindest/node:v1.31.0
```

**Multi-node cluster:**

```bash
# Create a cluster from a configuration file
kind create cluster --name multi-node --config kind-config.yaml
```

```yaml
# kind-config.yaml — multi-node cluster
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

### Cluster Configuration

kind uses a YAML configuration file to customize cluster behavior. Below are common configurations.

**Extra port mappings — expose services on the host:**

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30000
        hostPort: 30000
        protocol: TCP
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
```

**Extra mounts — mount host directories into nodes:**

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraMounts:
      - hostPath: /path/on/host
        containerPath: /data
        readOnly: false
        propagation: HostToContainer
```

**HA control plane with multiple workers:**

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: control-plane
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
```

### Loading Local Images

kind clusters run inside Docker containers and cannot pull images from the host Docker daemon. You must explicitly load images into the cluster.

```bash
# Build a local image
docker build -t my-app:latest .

# Load the image into a kind cluster
kind load docker-image my-app:latest --name my-app

# Load from an image archive
docker save my-app:latest -o my-app.tar
kind load image-archive my-app.tar --name my-app
```

> **Note:** Set `imagePullPolicy: Never` or `imagePullPolicy: IfNotPresent` in your Pod spec when using locally loaded images to prevent Kubernetes from attempting to pull from a remote registry.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: my-app
      image: my-app:latest
      imagePullPolicy: IfNotPresent
```

### Ingress Setup with kind

To use an Ingress controller with kind, map ports 80 and 443 from the control-plane node to the host, then deploy an Ingress controller.

**Step 1 — Create a cluster with port mappings and node labels:**

```yaml
# kind-ingress.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
```

```bash
kind create cluster --config kind-ingress.yaml
```

**Step 2 — Deploy the NGINX Ingress controller:**

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for the controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

**Step 3 — Create an Ingress resource:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
spec:
  rules:
    - host: my-app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-svc
                port:
                  number: 80
```

Add `127.0.0.1 my-app.local` to `/etc/hosts` (or `C:\Windows\System32\drivers\etc\hosts` on Windows) to access the application via the hostname.

### Persistent Storage

kind provides a default `StorageClass` called `standard` backed by a `rancher.io/local-path` provisioner. PersistentVolumeClaims are fulfilled automatically.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

For data that must survive cluster deletion, use `extraMounts` to bind a host directory into the node container and reference it in a `hostPath` PersistentVolume.

### Common kind Commands

```bash
# List clusters
kind get clusters

# Get cluster details and kubeconfig
kind get kubeconfig --name my-app

# Export kubeconfig to a file
kind get kubeconfig --name my-app > kubeconfig.yaml

# List node containers
kind get nodes --name my-app

# Delete a cluster
kind delete cluster --name my-app

# Delete all clusters
kind delete clusters --all

# Load an image into a cluster
kind load docker-image my-image:tag --name my-app

# Export logs for debugging
kind export logs ./kind-logs --name my-app
```

## Minikube

**Minikube** runs a single- or multi-node Kubernetes cluster inside a VM or container on the local machine. It supports multiple hypervisors and container runtimes and includes a rich set of addons for common Kubernetes extensions like Ingress, DNS, and the dashboard.

```
┌────────────────────────────────────────┐
│         Host Machine                   │
│  ┌──────────────────────────────────┐  │
│  │     Minikube VM / Container      │  │
│  │  ┌────────────┐ ┌────────────┐   │  │
│  │  │ kubelet    │ │ kube-proxy │   │  │
│  │  └────────────┘ └────────────┘   │  │
│  │  ┌────────────┐ ┌────────────┐   │  │
│  │  │ API server │ │    etcd    │   │  │
│  │  └────────────┘ └────────────┘   │  │
│  │  ┌────────────────────────────┐  │  │
│  │  │     Container Runtime      │  │  │
│  │  │  (docker / containerd /    │  │  │
│  │  │   cri-o)                   │  │  │
│  │  └────────────────────────────┘  │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
```

### Installing Minikube

**Linux:**

```bash
# Download and install the latest binary
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Verify
minikube version
```

**macOS:**

```bash
# Using Homebrew
brew install minikube

# Or download directly
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-amd64
sudo install minikube-darwin-amd64 /usr/local/bin/minikube
```

**Windows (PowerShell):**

```powershell
# Using Chocolatey
choco install minikube

# Or download the installer
Invoke-WebRequest -Uri "https://storage.googleapis.com/minikube/releases/latest/minikube-installer.exe" -OutFile "minikube-installer.exe"
Start-Process -Wait -FilePath ".\minikube-installer.exe"
```

### Drivers

Minikube supports multiple drivers that determine how the cluster node is provisioned. The driver should be chosen based on the host operating system and available hypervisor.

| Driver | Platform | Notes |
|---|---|---|
| `docker` | Linux, macOS, Windows | Recommended default — runs nodes as containers |
| `virtualbox` | Linux, macOS, Windows | Cross-platform VM driver; requires VirtualBox |
| `hyperkit` | macOS | Lightweight macOS-native hypervisor |
| `hyperv` | Windows | Built-in Windows hypervisor; requires Hyper-V enabled |
| `kvm2` | Linux | KVM-based driver for Linux; requires libvirt |
| `podman` | Linux | Rootless container runtime alternative to Docker |
| `none` | Linux | Runs directly on the host — no VM or container isolation |

```bash
# Start with a specific driver
minikube start --driver=docker

# Set a default driver
minikube config set driver docker
```

### Creating Clusters and Profiles

Minikube **profiles** allow you to run multiple isolated clusters simultaneously on the same machine. Each profile has its own configuration, Kubernetes version, and addons.

```bash
# Start the default cluster
minikube start

# Start with a specific Kubernetes version and resources
minikube start --kubernetes-version=v1.31.0 --cpus=4 --memory=8192

# Start a named profile
minikube start -p staging --kubernetes-version=v1.30.0
minikube start -p development --kubernetes-version=v1.31.0

# List all profiles
minikube profile list

# Switch the active profile
minikube profile staging

# Delete a profile
minikube delete -p staging
```

### Minikube Addons

Minikube ships with a curated set of addons that can be enabled with a single command.

```bash
# List all available addons and their status
minikube addons list
```

**Commonly used addons:**

| Addon | Purpose | Enable Command |
|---|---|---|
| `ingress` | NGINX Ingress controller | `minikube addons enable ingress` |
| `metrics-server` | Resource metrics for HPA and `kubectl top` | `minikube addons enable metrics-server` |
| `dashboard` | Kubernetes web dashboard | `minikube addons enable dashboard` |
| `registry` | In-cluster container image registry | `minikube addons enable registry` |
| `storage-provisioner` | Dynamic local PV provisioning (enabled by default) | — |
| `ingress-dns` | DNS resolution for Ingress hostnames | `minikube addons enable ingress-dns` |

```bash
# Enable ingress and metrics-server
minikube addons enable ingress
minikube addons enable metrics-server

# Open the dashboard in a browser
minikube dashboard

# Disable an addon
minikube addons disable dashboard
```

### Mounting Host Directories

Minikube can mount directories from the host into the cluster node, making local files accessible to Pods.

```bash
# Mount a host directory into the Minikube node
minikube mount /path/on/host:/mnt/data

# The mount runs in the foreground — use a separate terminal or background it
minikube mount /path/on/host:/mnt/data &
```

Then reference the mount path in a Pod using a `hostPath` volume:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dev-pod
spec:
  containers:
    - name: app
      image: my-app:latest
      volumeMounts:
        - name: host-data
          mountPath: /app/data
  volumes:
    - name: host-data
      hostPath:
        path: /mnt/data
        type: Directory
```

### LoadBalancer Services with minikube tunnel

Kubernetes `LoadBalancer` Services normally require a cloud provider to allocate an external IP. On Minikube, `minikube tunnel` creates a network route to expose LoadBalancer Services with a routable IP on the host.

```bash
# Run in a separate terminal — requires elevated privileges
minikube tunnel
```

```bash
# Create a LoadBalancer service
kubectl expose deployment my-app --type=LoadBalancer --port=80

# Check the external IP (populated while tunnel is running)
kubectl get svc my-app
# NAME     TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# my-app   LoadBalancer   10.96.45.12    10.96.45.12   80:31234/TCP   5s
```

> **Note:** `minikube tunnel` must remain running for LoadBalancer IPs to stay reachable. It may prompt for a password to configure network routes.

### Multi-Node Clusters

Minikube supports multi-node clusters for testing workload distribution, affinity rules, and node failure scenarios.

```bash
# Start a cluster with 3 nodes
minikube start --nodes=3

# Check node status
kubectl get nodes
# NAME           STATUS   ROLES           AGE   VERSION
# minikube       Ready    control-plane   60s   v1.31.0
# minikube-m02   Ready    <none>          30s   v1.31.0
# minikube-m03   Ready    <none>          15s   v1.31.0

# Add a node to an existing cluster
minikube node add

# Delete a specific node
minikube node delete minikube-m03
```

### Common Minikube Commands

```bash
# Start / stop / delete the cluster
minikube start
minikube stop
minikube delete

# Check cluster status
minikube status

# SSH into the Minikube node
minikube ssh

# Get the IP address of the cluster
minikube ip

# Access a NodePort service via minikube service
minikube service my-app --url

# View cluster logs
minikube logs

# Update Minikube itself
minikube update-check

# Configure the Docker CLI to use Minikube's Docker daemon
eval $(minikube docker-env)

# Reset Docker CLI to host daemon
eval $(minikube docker-env -u)
```

## Other Local Options

While kind and Minikube are the most common choices, several other tools provide local Kubernetes environments.

### k3d

**k3d** wraps Rancher's lightweight **k3s** distribution in Docker containers. It creates clusters quickly and uses fewer resources than full Kubernetes.

```bash
# Install k3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# Create a cluster
k3d cluster create my-cluster --servers 1 --agents 2

# Delete a cluster
k3d cluster delete my-cluster
```

### k3s

**k3s** is a CNCF-certified lightweight Kubernetes distribution designed for resource-constrained environments. It can run directly on the host without containers or VMs.

```bash
# Install k3s (Linux only)
curl -sfL https://get.k3s.io | sh -

# Check the cluster
sudo k3s kubectl get nodes
```

### MicroK8s

**MicroK8s** is Canonical's minimal Kubernetes distribution, installed via snap on Linux. It runs natively on the host and supports a wide range of addons.

```bash
# Install MicroK8s (Linux)
sudo snap install microk8s --classic

# Enable common addons
microk8s enable dns storage ingress

# Access kubectl
microk8s kubectl get nodes
```

### Docker Desktop Kubernetes

Docker Desktop includes a built-in single-node Kubernetes cluster that can be enabled in the settings. It is the simplest option but offers limited configuration and does not support multi-node clusters.

## Comparison Table

| Feature | kind | Minikube | k3d | Docker Desktop K8s |
|---|---|---|---|---|
| **Runtime** | Docker containers | VM or container | Docker containers | Built-in VM |
| **Multi-node** | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No |
| **HA control plane** | ✅ Yes | ❌ No | ✅ Yes | ❌ No |
| **Startup time** | ~30 seconds | ~60 seconds | ~20 seconds | ~60 seconds |
| **Resource usage** | Low | Medium–High | Low | Medium |
| **Addons** | Manual install | Built-in addon system | Manual install | Limited |
| **CI/CD suitability** | ✅ Excellent | ⚠️ Possible | ✅ Excellent | ❌ Poor |
| **Image loading** | `kind load` | `minikube image load` or shared daemon | `k3d image import` | Shared daemon |
| **LoadBalancer support** | ❌ Manual | ✅ `minikube tunnel` | ✅ Built-in | ✅ Built-in |
| **GPU support** | ❌ No | ✅ Yes | ❌ No | ❌ No |
| **Ingress** | Manual deploy | Addon | Manual deploy | Manual deploy |
| **Primary use case** | CI testing, development | Development, learning | CI testing, development | Quick development |

## Development Workflow with Local Clusters

### Building and Testing Container Images Locally

A typical inner-loop workflow involves building container images on the developer machine and deploying them into the local cluster without pushing to a remote registry.

**Workflow with kind:**

```bash
# 1. Build the image
docker build -t my-app:dev .

# 2. Load into kind
kind load docker-image my-app:dev --name my-cluster

# 3. Apply manifests
kubectl apply -f k8s/

# 4. Verify
kubectl get pods
kubectl logs deployment/my-app
```

**Workflow with Minikube (shared Docker daemon):**

```bash
# 1. Point Docker CLI to Minikube's daemon
eval $(minikube docker-env)

# 2. Build the image — it is now available inside Minikube
docker build -t my-app:dev .

# 3. Apply manifests (use imagePullPolicy: Never)
kubectl apply -f k8s/

# 4. Reset Docker CLI when done
eval $(minikube docker-env -u)
```

### Hot-Reload and Live Development

Manual build-load-deploy cycles are slow. The following tools automate the inner loop by watching for file changes and synchronizing code or rebuilt images into the cluster.

**Skaffold:**

```bash
# Install Skaffold
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
chmod +x skaffold && sudo mv skaffold /usr/local/bin/

# Initialize in your project
skaffold init

# Start continuous development
skaffold dev
```

Skaffold watches source files, rebuilds images, and redeploys to the cluster automatically.

**Tilt:**

```bash
# Install Tilt
curl -fsSL https://raw.githubusercontent.com/tilt-dev/tilt/master/scripts/install.sh | bash

# Start Tilt (reads Tiltfile in the project root)
tilt up
```

Tilt provides a web UI showing build status, pod logs, and resource health in real time.

**Telepresence:**

Telepresence creates a bidirectional network proxy between the local machine and the cluster, allowing you to run a service locally while it communicates with in-cluster services.

```bash
# Install Telepresence
sudo curl -fL https://app.getambassador.io/download/tel2oss/releases/download/v2.18.0/telepresence-linux-amd64 \
  -o /usr/local/bin/telepresence
sudo chmod +x /usr/local/bin/telepresence

# Connect to the cluster
telepresence connect

# Intercept a service — route its traffic to your local process
telepresence intercept my-app --port 8080:80
```

## Troubleshooting Common Issues

| Problem | Cause | Solution |
|---|---|---|
| `kind create cluster` fails | Docker daemon not running | Start Docker: `sudo systemctl start docker` |
| Image not found in kind cluster | Image not loaded into cluster | Run `kind load docker-image <image> --name <cluster>` |
| `ImagePullBackOff` with local images | `imagePullPolicy` set to `Always` | Set `imagePullPolicy: IfNotPresent` or `Never` |
| Minikube fails to start | Insufficient resources | Increase: `minikube start --cpus=4 --memory=8192` |
| NodePort not reachable on Minikube | VM networking | Use `minikube service <name> --url` to get the correct URL |
| LoadBalancer stuck in `<pending>` (Minikube) | Tunnel not running | Run `minikube tunnel` in a separate terminal |
| Pods stuck in `Pending` | No worker nodes or insufficient resources | Check `kubectl describe pod <name>` for scheduling errors |
| DNS resolution fails in cluster | CoreDNS not running | Verify: `kubectl get pods -n kube-system -l k8s-app=kube-dns` |
| kind cluster unreachable after Docker restart | Docker network recreated | Delete and recreate the cluster: `kind delete cluster && kind create cluster` |
| Disk pressure on Minikube node | Disk full inside VM | Increase disk: `minikube start --disk-size=40g` or prune images: `minikube ssh -- docker system prune -a` |

**Collecting diagnostic information:**

```bash
# kind — export logs to a directory
kind export logs ./kind-debug --name my-cluster

# Minikube — view logs
minikube logs --file=minikube-debug.log

# General — inspect cluster events
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl describe node
```

## Best Practices

### Cluster Management

- ✅ Use named clusters (`kind create cluster --name my-app`) to avoid conflicts between projects
- ✅ Pin Kubernetes versions in CI to ensure reproducible test results
- ✅ Delete clusters after use to free Docker resources — `kind delete cluster` or `minikube delete`
- ❌ Do not leave idle clusters running — they consume CPU and memory even when unused

### Image Management

- ✅ Use specific image tags (`:dev`, `:v1.2.3`) — avoid `:latest` to prevent caching issues
- ✅ Set `imagePullPolicy: IfNotPresent` for locally loaded images
- ❌ Do not push development images to production registries

### Resource Allocation

- ✅ Allocate at least 2 CPUs and 4 GB of memory for single-node clusters
- ✅ Allocate at least 4 CPUs and 8 GB of memory for multi-node clusters
- ✅ Monitor host resource usage — local clusters share resources with other applications

### CI/CD Integration

- ✅ Prefer kind or k3d for CI — they start faster and are designed for ephemeral use
- ✅ Use configuration files checked into version control for reproducible cluster setup
- ❌ Do not use Docker Desktop Kubernetes in CI — it is not designed for automation

### Security

- ✅ Treat local clusters as disposable — never store production secrets in them
- ✅ Use separate kubeconfig files per cluster to avoid accidental operations on production
- ❌ Do not expose local cluster ports to public networks

## Next Steps

Continue exploring Kubernetes topics with the other documents in this series:

- [Kubernetes Overview](00-OVERVIEW.md) — Architecture and core concepts
- [Kubernetes Services](01-SERVICES.md) — Service types, discovery, and networking
- [Cloud Services — AWS EKS](02-CLOUD-SERVICES-AWS-EKS.md) — Deploying to managed Kubernetes on AWS
- [Cloud Services — OKE](03-CLOUD-SERVICES-OKE.md) — Deploying to Oracle Kubernetes Engine
- [Cloud Services — AKS](04-CLOUD-SERVICES-AKS.md) — Deploying to Azure Kubernetes Service

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial local Kubernetes setup documentation |
