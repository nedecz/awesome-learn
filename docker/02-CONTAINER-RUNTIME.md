# Container Runtime

## Table of Contents

1. [Overview](#overview)
   - [What Is a Container Runtime](#what-is-a-container-runtime)
   - [The OCI Ecosystem](#the-oci-ecosystem)
2. [Docker Engine Architecture](#docker-engine-architecture)
   - [dockerd](#dockerd)
   - [containerd](#containerd-overview)
   - [runc](#runc-overview)
   - [containerd-shim](#containerd-shim)
3. [High-Level vs Low-Level Runtimes](#high-level-vs-low-level-runtimes)
4. [Container Lifecycle Management](#container-lifecycle-management)
   - [Creating Containers](#creating-containers)
   - [Starting Containers](#starting-containers)
   - [Stopping Containers](#stopping-containers)
   - [Killing Containers](#killing-containers)
   - [Removing Containers](#removing-containers)
   - [Complete Lifecycle Example](#complete-lifecycle-example)
5. [containerd Deep Dive](#containerd-deep-dive)
   - [Architecture](#containerd-architecture)
   - [Namespaces](#containerd-namespaces)
   - [Snapshots](#snapshots)
   - [Content Store](#content-store)
6. [runc and the OCI Runtime Spec](#runc-and-the-oci-runtime-spec)
   - [OCI Runtime Specification](#oci-runtime-specification)
   - [config.json](#configjson)
   - [Running a Container with runc](#running-a-container-with-runc)
7. [Alternative Runtimes](#alternative-runtimes)
   - [CRI-O](#cri-o)
   - [Kata Containers](#kata-containers)
   - [gVisor](#gvisor)
   - [Runtime Comparison](#runtime-comparison)
8. [Container Runtime Interface (CRI)](#container-runtime-interface-cri)
   - [CRI Architecture](#cri-architecture)
   - [CRI in Kubernetes](#cri-in-kubernetes)
   - [Migrating from Docker to containerd](#migrating-from-docker-to-containerd)
9. [Debugging Containers](#debugging-containers)
   - [docker inspect](#docker-inspect)
   - [docker logs](#docker-logs)
   - [docker exec](#docker-exec)
   - [nsenter](#nsenter)
   - [Debugging Checklist](#debugging-checklist)
10. [Next Steps](#next-steps)
11. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **container runtimes** — the software responsible for actually running containers on a host. It covers the Docker Engine architecture, the OCI specifications that standardize container behavior, and alternative runtimes used in production Kubernetes clusters and specialized environments.

### Target Audience

- **DevOps Engineers** managing container infrastructure and runtime configuration
- **Platform Engineers** selecting and configuring container runtimes for Kubernetes clusters
- **SREs** debugging container behavior at the runtime level
- **Developers** who want to understand what happens beneath `docker run`

### Scope

- Docker Engine components: dockerd, containerd, runc, and containerd-shim
- High-level and low-level container runtime classifications
- Container lifecycle management with practical commands
- containerd internals: namespaces, snapshots, and content store
- OCI Runtime Specification and runc
- Alternative runtimes: CRI-O, Kata Containers, gVisor
- Container Runtime Interface (CRI) for Kubernetes
- Debugging techniques at the runtime level

### What Is a Container Runtime

A **container runtime** is the software that executes containers on a host operating system. It is responsible for:

- Pulling and unpacking container images
- Creating isolated environments using Linux kernel primitives (namespaces, cgroups)
- Managing the container lifecycle (create, start, stop, kill, remove)
- Attaching storage and networking to containers

Container runtimes exist at two levels:

```
┌────────────────────────────────────────────────────────────┐
│              HIGH-LEVEL RUNTIME                              │
│         (containerd, CRI-O, dockerd)                        │
│                                                            │
│   Image management, API, networking, storage,              │
│   container supervision, garbage collection                │
│                                                            │
│   ┌──────────────────────────────────────────────────────┐ │
│   │           LOW-LEVEL RUNTIME                           │ │
│   │          (runc, crun, kata, gVisor)                   │ │
│   │                                                      │ │
│   │   Create namespaces, set cgroups, pivot_root,        │ │
│   │   exec container process, handle signals             │ │
│   └──────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

### The OCI Ecosystem

The **Open Container Initiative (OCI)** defines three specifications that ensure interoperability across the container ecosystem:

| Specification | Purpose | Key Contents |
|---|---|---|
| **Runtime Spec** | How to run a container | Filesystem bundle format, `config.json` schema, lifecycle operations |
| **Image Spec** | How to package a container image | Image manifest, layer format, configuration, content-addressable storage |
| **Distribution Spec** | How to distribute images | Registry API for push/pull, content discovery, tag listing |

These specifications allow any OCI-compliant image to run on any OCI-compliant runtime, regardless of which tool built the image.

---

## Docker Engine Architecture

The Docker Engine is composed of several cooperating components, each with a clear responsibility:

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Docker Client (CLI)                          │
│                                                                      │
│  $ docker run    $ docker build    $ docker ps    $ docker logs      │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             │  gRPC / REST API
                             │  (via /var/run/docker.sock)
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        dockerd (Docker Daemon)                        │
│                                                                      │
│   • Exposes the Docker API                                           │
│   • Manages images, networks, volumes                                │
│   • Orchestrates builds                                              │
│   • Delegates container operations to containerd                     │
│                                                                      │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
                             │  gRPC API
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                          containerd                                    │
│                                                                      │
│   • Image pull / push / unpack                                       │
│   • Snapshot management (filesystem layers)                          │
│   • Container lifecycle management                                   │
│   • Task execution (via shims)                                       │
│                                                                      │
│   ┌──────────────────────────────────────────────────────────────┐   │
│   │                    containerd-shim-runc-v2                    │   │
│   │                                                              │   │
│   │   • Per-container process                                    │   │
│   │   • Allows containerd/dockerd restart without killing        │   │
│   │     running containers                                       │   │
│   │   • Handles stdin/stdout/stderr forwarding                   │   │
│   │   • Reports exit status back to containerd                   │   │
│   │                                                              │   │
│   └──────────────────────────┬───────────────────────────────────┘   │
│                              │                                       │
│                              │  fork/exec                            │
│                              ▼                                       │
│                    ┌──────────────────┐                               │
│                    │      runc        │                               │
│                    │                  │                               │
│                    │  • OCI Runtime   │                               │
│                    │  • Configures    │                               │
│                    │    namespaces    │                               │
│                    │  • Sets cgroups  │                               │
│                    │  • Spawns init   │                               │
│                    │    process       │                               │
│                    │  • Exits after   │                               │
│                    │    container     │                               │
│                    │    starts        │                               │
│                    └──────────────────┘                               │
└──────────────────────────────────────────────────────────────────────┘
```

### dockerd

The Docker daemon (`dockerd`) is the long-running process that listens for Docker API requests. It is responsible for:

- Exposing the Docker REST/gRPC API on a Unix socket or TCP port
- Managing Docker objects: images, containers, networks, volumes, and plugins
- Orchestrating image builds (BuildKit integration)
- Authenticating with container registries
- Delegating all container execution operations to containerd

```bash
# Check dockerd status
systemctl status docker

# View dockerd configuration
cat /etc/docker/daemon.json

# Example daemon.json
# {
#   "storage-driver": "overlay2",
#   "log-driver": "json-file",
#   "log-opts": { "max-size": "10m", "max-file": "3" },
#   "default-address-pools": [
#     { "base": "172.17.0.0/16", "size": 24 }
#   ]
# }
```

### containerd Overview

**containerd** is a CNCF Graduated project that provides a complete container runtime. Docker delegates all container operations to containerd, but containerd can also be used independently (as it is in many Kubernetes deployments).

```bash
# Check containerd status
systemctl status containerd

# Use ctr (containerd CLI) to interact directly
ctr --namespace moby containers list
ctr --namespace moby images list

# containerd configuration
cat /etc/containerd/config.toml
```

### runc Overview

**runc** is the reference implementation of the OCI Runtime Specification. It is a lightweight CLI tool that spawns and runs containers according to the OCI spec. runc is invoked by containerd-shim and exits after the container process starts.

```bash
# Check runc version
runc --version
# runc version 1.1.12
# spec: 1.0.2-dev

# runc can be used standalone (advanced usage)
# 1. Export a container filesystem
mkdir rootfs
docker export $(docker create alpine) | tar -C rootfs -xf -

# 2. Generate an OCI spec
runc spec

# 3. Run the container
runc run my-container
```

### containerd-shim

The **containerd-shim** is a small process that sits between containerd and the actual container process. It provides critical decoupling:

- **Daemon restarts** — containerd or dockerd can be restarted without killing running containers
- **File descriptor management** — keeps stdin/stdout/stderr open for the container
- **Exit status** — reports container exit status back to containerd
- **One shim per container** — each container gets its own shim process

```
Process Tree:
  systemd
  ├── dockerd
  │   └── containerd
  │       ├── containerd-shim-runc-v2 ──► container process (PID 1 in namespace)
  │       ├── containerd-shim-runc-v2 ──► container process
  │       └── containerd-shim-runc-v2 ──► container process
```

---

## High-Level vs Low-Level Runtimes

| Feature | containerd | CRI-O | runc | crun | Kata Containers | gVisor (runsc) |
|---|---|---|---|---|---|---|
| **Level** | High | High | Low | Low | Low | Low |
| **Image management** | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Container supervision** | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Networking** | ✅ (CNI) | ✅ (CNI) | ❌ | ❌ | ❌ | ❌ |
| **Storage/snapshots** | ✅ | ✅ (limited) | ❌ | ❌ | ❌ | ❌ |
| **OCI compliant** | ✅ | ✅ | ✅ (reference) | ✅ | ✅ | ✅ |
| **Kubernetes CRI** | ✅ (built-in) | ✅ (native) | ❌ | ❌ | Via shimv2 | Via shimv2 |
| **Isolation** | Namespace/cgroup | Namespace/cgroup | Namespace/cgroup | Namespace/cgroup | MicroVM | User-space kernel |
| **Performance overhead** | Minimal | Minimal | Minimal | Lower than runc | Higher (VM boot) | Moderate |
| **Written in** | Go | Go | Go | C | Go/Rust | Go |
| **Use case** | General purpose | Kubernetes-only | Default low-level | Performance-critical | Strong isolation | Untrusted workloads |

---

## Container Lifecycle Management

A container passes through well-defined states during its lifetime:

```
                         docker create
                              │
                              ▼
                       ┌─────────────┐
                       │   Created    │
                       └──────┬──────┘
                              │ docker start
                              ▼
                       ┌─────────────┐     docker pause     ┌──────────┐
                       │   Running    │────────────────────►│  Paused   │
                       │              │◄────────────────────│          │
                       └──────┬──────┘    docker unpause    └──────────┘
                              │
               ┌──────────────┼──────────────┐
               │              │              │
         docker stop    docker kill    process exits
               │              │              │
               ▼              ▼              ▼
        ┌─────────────────────────────────────────┐
        │              Exited / Stopped            │
        └──────────────────┬──────────────────────┘
                           │
                    docker rm (or docker start to restart)
                           │
                           ▼
                    ┌─────────────┐
                    │   Removed    │
                    └─────────────┘
```

### Creating Containers

`docker create` creates a container from an image but does not start it. The container is in the **Created** state.

```bash
# Create a container without starting it
docker create --name my-nginx -p 8080:80 nginx:1.25

# Create with environment variables and resource limits
docker create \
    --name my-app \
    --env NODE_ENV=production \
    --memory 512m \
    --cpus 1.5 \
    -p 3000:3000 \
    myapp:1.0

# Verify the container is created
docker ps -a --filter name=my-app
# STATUS: Created
```

### Starting Containers

`docker start` transitions a container from **Created** (or **Exited**) to **Running**.

```bash
# Start a created container
docker start my-nginx

# Start and attach to stdout/stderr
docker start -a my-app

# The shortcut: docker run = docker create + docker start
docker run -d --name web -p 8080:80 nginx:1.25
```

### Stopping Containers

`docker stop` sends SIGTERM to the container's PID 1 process and waits for a grace period (default 10 seconds) before sending SIGKILL.

```bash
# Stop with default 10-second grace period
docker stop my-nginx

# Stop with custom grace period
docker stop --time 30 my-app

# Stop all running containers
docker stop $(docker ps -q)
```

### Killing Containers

`docker kill` sends a signal (default SIGKILL) to the container's PID 1 immediately, without a grace period.

```bash
# Kill with SIGKILL (default)
docker kill my-nginx

# Kill with a specific signal
docker kill --signal SIGTERM my-app
docker kill --signal SIGHUP my-nginx
```

### Removing Containers

`docker rm` removes a stopped container. Use `-f` to force-remove a running container (sends SIGKILL first).

```bash
# Remove a stopped container
docker rm my-nginx

# Force remove a running container
docker rm -f my-app

# Remove all stopped containers
docker container prune -f

# Remove all stopped containers (alternative)
docker rm $(docker ps -a -q --filter status=exited)
```

### Complete Lifecycle Example

```bash
# 1. Create
docker create --name lifecycle-demo -p 8080:80 nginx:1.25
echo "State: Created"
docker inspect --format '{{.State.Status}}' lifecycle-demo

# 2. Start
docker start lifecycle-demo
echo "State: Running"
docker inspect --format '{{.State.Status}}' lifecycle-demo

# 3. Pause
docker pause lifecycle-demo
echo "State: Paused"
docker inspect --format '{{.State.Status}}' lifecycle-demo

# 4. Unpause
docker unpause lifecycle-demo
echo "State: Running (unpaused)"
docker inspect --format '{{.State.Status}}' lifecycle-demo

# 5. Stop (SIGTERM + grace period)
docker stop --time 5 lifecycle-demo
echo "State: Exited"
docker inspect --format '{{.State.Status}}' lifecycle-demo

# 6. Restart
docker start lifecycle-demo
echo "State: Running (restarted)"

# 7. Kill (SIGKILL — immediate)
docker kill lifecycle-demo
echo "State: Exited (killed)"

# 8. Remove
docker rm lifecycle-demo
echo "State: Removed"
```

---

## containerd Deep Dive

### containerd Architecture

containerd is organized as a set of plugins around a core:

```
┌──────────────────────────────────────────────────────────────────┐
│                         containerd                                │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐│
│  │   Content   │  │  Snapshots │  │   Images   │  │  Containers││
│  │   Store     │  │            │  │            │  │            ││
│  │            │  │ overlayfs  │  │  manifest  │  │  metadata  ││
│  │ blobs,     │  │ btrfs      │  │  index     │  │  spec      ││
│  │ manifests  │  │ devmapper  │  │  platform  │  │  state     ││
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘│
│        │               │               │               │        │
│  ┌─────┴───────────────┴───────────────┴───────────────┴──────┐ │
│  │                       Metadata Store                        │ │
│  │                    (bbolt key-value DB)                     │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌──────────────────────┐  ┌──────────────────────────────────┐ │
│  │    Task Service       │  │       Events / Metrics           │ │
│  │                      │  │                                  │ │
│  │  Create, start,      │  │  Publish lifecycle events        │ │
│  │  stop, kill tasks    │  │  Expose Prometheus metrics       │ │
│  │  via runtime shims   │  │                                  │ │
│  └──────────────────────┘  └──────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### containerd Namespaces

containerd uses **namespaces** (not to be confused with Linux kernel namespaces) to isolate resources within a single containerd instance.

```bash
# Docker uses the "moby" namespace
ctr namespaces list
# NAME   LABELS
# moby
# k8s.io

# List containers in the moby namespace (Docker)
ctr --namespace moby containers list

# List containers in the k8s.io namespace (Kubernetes)
ctr --namespace k8s.io containers list

# Create a custom namespace
ctr namespaces create my-namespace
```

| Namespace | Used By | Contains |
|---|---|---|
| `moby` | Docker Engine | All Docker-managed containers and images |
| `k8s.io` | Kubernetes (kubelet) | All Kubernetes Pod containers and images |
| Custom | Direct containerd users | User-defined workloads via `ctr` or `nerdctl` |

### Snapshots

Snapshots are containerd's mechanism for managing filesystem layers. Each image layer becomes a snapshot, and container root filesystems are built by stacking snapshots.

```bash
# List snapshotter plugins
ctr plugins ls | grep snapshot

# List snapshots in the moby namespace
ctr --namespace moby snapshots ls

# Inspect a specific snapshot
ctr --namespace moby snapshots info <snapshot-key>
```

| Snapshotter | Description | Use Case |
|---|---|---|
| `overlayfs` | Default on most Linux distributions | General purpose; best compatibility |
| `native` | Simple copy-based snapshotter | Testing or when overlayfs is unavailable |
| `btrfs` | Uses btrfs subvolumes and snapshots | Hosts with btrfs filesystem |
| `devmapper` | Uses device mapper thin provisioning | High-performance block-level storage |
| `zfs` | Uses ZFS clones | Hosts with ZFS filesystem |

### Content Store

The content store is containerd's content-addressable storage for image data — manifests, configs, and layer blobs.

```bash
# List content in the store
ctr --namespace moby content ls

# Fetch content by digest
ctr --namespace moby content get sha256:abc123...

# View image manifest
ctr --namespace moby images list
ctr --namespace moby images check
```

```
Content Store Layout:
  /var/lib/containerd/io.containerd.content.v1.content/
  ├── blobs/
  │   └── sha256/
  │       ├── a1b2c3...  (image manifest)
  │       ├── d4e5f6...  (image config)
  │       ├── 789abc...  (layer tar.gz)
  │       └── def012...  (layer tar.gz)
  └── ingest/             (temporary directory for in-progress pulls)
```

---

## runc and the OCI Runtime Spec

### OCI Runtime Specification

The OCI Runtime Specification defines how to run a **filesystem bundle** — a directory containing a `config.json` and a root filesystem. The spec covers:

- **Lifecycle operations** — create, start, kill, delete, state
- **Configuration** — namespaces, cgroups, mounts, process settings, hooks
- **Linux-specific features** — seccomp, AppArmor, SELinux, capabilities

### config.json

The `config.json` file describes the container's runtime configuration:

```json
{
    "ociVersion": "1.0.2",
    "process": {
        "terminal": false,
        "user": { "uid": 0, "gid": 0 },
        "args": ["sh"],
        "env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "TERM=xterm"
        ],
        "cwd": "/"
    },
    "root": {
        "path": "rootfs",
        "readonly": true
    },
    "linux": {
        "namespaces": [
            { "type": "pid" },
            { "type": "network" },
            { "type": "ipc" },
            { "type": "uts" },
            { "type": "mount" }
        ],
        "resources": {
            "memory": { "limit": 536870912 },
            "cpu": { "shares": 1024 }
        }
    }
}
```

### Running a Container with runc

```bash
# Step 1: Create a filesystem bundle
mkdir -p my-container/rootfs
docker export $(docker create alpine:3.19) | tar -C my-container/rootfs -xf -

# Step 2: Generate a default OCI spec
cd my-container
runc spec

# Step 3: Create the container (does not start it)
runc create my-container

# Step 4: Start the container
runc start my-container

# Step 5: Check state
runc state my-container

# Step 6: Kill and delete
runc kill my-container SIGTERM
runc delete my-container
```

---

## Alternative Runtimes

### CRI-O

**CRI-O** is a lightweight container runtime specifically designed for Kubernetes. It implements the Kubernetes Container Runtime Interface (CRI) and nothing more — no Docker API, no build tools, no `docker` CLI.

```bash
# CRI-O installation (Fedora/CentOS)
dnf install -y cri-o
systemctl enable --now crio

# Verify CRI-O status
crictl --runtime-endpoint unix:///var/run/crio/crio.sock info

# List containers via crictl
crictl ps
crictl pods
```

**Key characteristics:**
- Purpose-built for Kubernetes (implements CRI only)
- Supports OCI images and OCI runtimes
- Lighter than containerd for Kubernetes-only use cases
- Version-locked to Kubernetes releases (CRI-O 1.29 for Kubernetes 1.29)

### Kata Containers

**Kata Containers** run each container inside a lightweight virtual machine (microVM), providing hardware-level isolation while maintaining the container experience.

```
┌────────────────────────────────────────────────────┐
│                    Host Kernel                       │
│                                                    │
│  ┌──────────────────┐  ┌──────────────────┐        │
│  │    Kata MicroVM   │  │    Kata MicroVM   │        │
│  │  ┌────────────┐  │  │  ┌────────────┐  │        │
│  │  │  Guest     │  │  │  │  Guest     │  │        │
│  │  │  Kernel    │  │  │  │  Kernel    │  │        │
│  │  │  ┌──────┐  │  │  │  │  ┌──────┐  │  │        │
│  │  │  │ App  │  │  │  │  │  │ App  │  │  │        │
│  │  │  └──────┘  │  │  │  │  └──────┘  │  │        │
│  │  └────────────┘  │  │  └────────────┘  │        │
│  └──────────────────┘  └──────────────────┘        │
└────────────────────────────────────────────────────┘
```

**Use cases:** Multi-tenant environments, untrusted workloads, compliance requirements that mandate VM-level isolation.

### gVisor

**gVisor** (runsc) is a user-space kernel written in Go that intercepts application system calls and implements them in a sandboxed environment, without delegating to the host kernel.

```
Standard Container:                gVisor Container:
┌─────────────────────┐           ┌─────────────────────┐
│   Application        │           │   Application        │
│         │            │           │         │            │
│    system calls      │           │    system calls      │
│         │            │           │         │            │
│         ▼            │           │         ▼            │
│   ┌───────────┐      │           │   ┌───────────┐      │
│   │ Host      │      │           │   │  Sentry   │      │
│   │ Kernel    │      │           │   │ (gVisor   │      │
│   │           │      │           │   │  kernel)  │      │
│   └───────────┘      │           │   └─────┬─────┘      │
└─────────────────────┘           │         │ limited    │
                                  │         │ syscalls   │
  Full kernel access              │         ▼            │
  (~300+ syscalls)                │   ┌───────────┐      │
                                  │   │ Host      │      │
                                  │   │ Kernel    │      │
                                  │   └───────────┘      │
                                  └─────────────────────┘
                                    Reduced attack surface
                                    (~60 syscalls to host)
```

**Use cases:** Running untrusted code, serverless platforms, defense-in-depth security.

### Runtime Comparison

| Feature | runc | crun | Kata Containers | gVisor (runsc) |
|---|---|---|---|---|
| **Isolation model** | Namespaces + cgroups | Namespaces + cgroups | Hardware VM | User-space kernel |
| **Security boundary** | Linux kernel | Linux kernel | Hypervisor + guest kernel | gVisor Sentry |
| **Startup latency** | ~100ms | ~50ms | ~500ms–1s | ~200ms |
| **Memory overhead** | Minimal (~5 MB) | Minimal (~3 MB) | ~30–100 MB (VM) | ~15–30 MB |
| **Syscall compatibility** | Full | Full | Full (guest kernel) | Partial (~70%) |
| **Performance** | Near-native | Near-native | Near-native (with VT-x) | 10–30% overhead |
| **Language** | Go | C | Go / Rust | Go |
| **Best for** | General workloads | Performance-sensitive | Strong isolation needs | Untrusted code |

---

## Container Runtime Interface (CRI)

### CRI Architecture

The **Container Runtime Interface (CRI)** is a Kubernetes plugin API that allows the kubelet to use different container runtimes without code changes. CRI defines two gRPC services:

```
┌────────────────────────────────────────────────────────────────┐
│                          kubelet                                │
│                                                                │
│   ┌──────────────────────┐    ┌──────────────────────────┐    │
│   │  RuntimeService      │    │  ImageService             │    │
│   │                      │    │                          │    │
│   │  • RunPodSandbox     │    │  • PullImage             │    │
│   │  • StopPodSandbox    │    │  • ListImages            │    │
│   │  • CreateContainer   │    │  • RemoveImage           │    │
│   │  • StartContainer    │    │  • ImageStatus           │    │
│   │  • StopContainer     │    │  • ImageFsInfo           │    │
│   │  • RemoveContainer   │    │                          │    │
│   │  • ExecSync          │    └──────────────────────────┘    │
│   │  • Attach            │                                    │
│   │  • PortForward       │                                    │
│   └──────────┬───────────┘                                    │
│              │  gRPC                                          │
└──────────────┼────────────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────┐
│     CRI Runtime              │
│  (containerd / CRI-O)        │
│              │                │
│              ▼                │
│     OCI Runtime              │
│  (runc / crun / kata / runsc)│
└──────────────────────────────┘
```

### CRI in Kubernetes

Since Kubernetes 1.24, the **dockershim** has been removed. Kubernetes now communicates directly with CRI-compliant runtimes.

| Kubernetes Version | Docker Support | Default Runtime |
|---|---|---|
| ≤ 1.23 | Via dockershim (built-in) | Docker Engine |
| 1.24+ | Via cri-dockerd (external) | containerd |

```bash
# Check which runtime your kubelet is using
kubectl get nodes -o wide
# CONTAINER-RUNTIME column shows: containerd://1.7.x or cri-o://1.29.x

# Verify the CRI endpoint
crictl --runtime-endpoint unix:///run/containerd/containerd.sock info
```

### Migrating from Docker to containerd

```bash
# 1. Drain the node
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 2. Stop Docker and kubelet
systemctl stop kubelet
systemctl stop docker

# 3. Configure containerd
containerd config default > /etc/containerd/config.toml
# Edit SystemdCgroup = true if using systemd cgroup driver

# 4. Update kubelet flags
# Set --container-runtime-endpoint=unix:///run/containerd/containerd.sock

# 5. Restart services
systemctl restart containerd
systemctl restart kubelet

# 6. Uncordon the node
kubectl uncordon <node-name>

# 7. Verify
kubectl get node <node-name> -o wide
# CONTAINER-RUNTIME: containerd://1.7.x
```

---

## Debugging Containers

### docker inspect

`docker inspect` returns detailed JSON metadata about a container, including configuration, state, networking, and mounts.

```bash
# Full inspect output
docker inspect my-container

# Get container IP address
docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-container

# Get container state
docker inspect --format '{{.State.Status}}' my-container

# Get environment variables
docker inspect --format '{{json .Config.Env}}' my-container | jq .

# Get mount points
docker inspect --format '{{json .Mounts}}' my-container | jq .

# Get restart count
docker inspect --format '{{.RestartCount}}' my-container

# Get OOM killed status
docker inspect --format '{{.State.OOMKilled}}' my-container
```

### docker logs

`docker logs` retrieves the stdout and stderr output from a container.

```bash
# View all logs
docker logs my-container

# Follow logs in real time
docker logs -f my-container

# Show last 100 lines
docker logs --tail 100 my-container

# Show logs since a timestamp
docker logs --since 2025-01-15T10:00:00Z my-container

# Show logs with timestamps
docker logs -t my-container

# Combine options: last 50 lines with follow and timestamps
docker logs -f --tail 50 -t my-container
```

### docker exec

`docker exec` runs a new command inside a running container.

```bash
# Open an interactive shell
docker exec -it my-container /bin/sh

# Run a specific command
docker exec my-container cat /etc/os-release

# Run as a specific user
docker exec -u root my-container whoami

# Set environment variables for the exec session
docker exec -e DEBUG=true my-container printenv DEBUG

# Run in a specific working directory
docker exec -w /app my-container ls -la
```

### nsenter

`nsenter` is a host-level tool that enters the namespaces of a running container. Useful when the container has no shell or debugging tools.

```bash
# Find the container's PID on the host
CONTAINER_PID=$(docker inspect --format '{{.State.Pid}}' my-container)

# Enter all namespaces of the container
nsenter -t $CONTAINER_PID -m -u -i -n -p -- /bin/sh

# Enter only the network namespace (useful for network debugging)
nsenter -t $CONTAINER_PID -n -- ip addr show
nsenter -t $CONTAINER_PID -n -- ss -tlnp
nsenter -t $CONTAINER_PID -n -- ping 10.0.0.1

# Enter only the mount namespace (see container filesystem)
nsenter -t $CONTAINER_PID -m -- ls /app
```

### Debugging Checklist

| Symptom | Tool | Command |
|---|---|---|
| Container won't start | `docker inspect` | `docker inspect --format '{{.State.Error}}' <container>` |
| Application crashing | `docker logs` | `docker logs --tail 200 <container>` |
| Out of memory | `docker inspect` | `docker inspect --format '{{.State.OOMKilled}}' <container>` |
| High CPU / memory | `docker stats` | `docker stats <container>` |
| Network not reachable | `docker exec` / `nsenter` | `docker exec <container> ping <target>` |
| Filesystem issues | `docker exec` / `docker diff` | `docker diff <container>` |
| Container is running but unresponsive | `docker top` | `docker top <container>` |
| Need host-level debugging | `nsenter` | `nsenter -t <PID> -n -- ss -tlnp` |

---

## Next Steps

Continue your Docker learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Docker Overview | Core Docker concepts, architecture, and CLI essentials |
| [01-IMAGES-AND-DOCKERFILES.md](01-IMAGES-AND-DOCKERFILES.md) | Images & Dockerfiles | Building images, Dockerfile instructions, multi-stage builds |
| [03-NETWORKING.md](03-NETWORKING.md) | Docker Networking | Bridge, host, overlay networks, DNS, and port mapping |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Container Runtime documentation |
