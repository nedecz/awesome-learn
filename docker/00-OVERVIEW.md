# Docker & Containers Overview

## Table of Contents

1. [Overview](#overview)
2. [What are Containers](#what-are-containers)
3. [Docker Architecture](#docker-architecture)
4. [Container Fundamentals](#container-fundamentals)
5. [OCI Standards](#oci-standards)
6. [Docker CLI Essentials](#docker-cli-essentials)
7. [Container Lifecycle](#container-lifecycle)
8. [Docker Desktop vs Docker Engine](#docker-desktop-vs-docker-engine)
9. [Prerequisites](#prerequisites)
10. [Next Steps](#next-steps)
11. [Version History](#version-history)

---

## Overview

This documentation provides a comprehensive introduction to Docker and container technology. It covers the foundational concepts, architecture, tooling, and practical examples needed to build, ship, and run containerized applications.

### Target Audience

- **Developers** building and packaging applications into portable container images
- **DevOps Engineers** designing CI/CD pipelines that build, test, and deploy containers
- **Site Reliability Engineers (SREs)** operating container workloads in production and troubleshooting runtime issues
- **Architects** evaluating containerization strategies and container runtime choices

### Scope

- What containers are and how they compare to virtual machines
- Docker architecture: client, daemon, containerd, and runc
- Linux kernel primitives that make containers possible (namespaces, cgroups, union filesystems)
- OCI standards for images, runtimes, and distribution
- Essential Docker CLI commands for day-to-day use
- Container lifecycle management
- Docker Desktop vs Docker Engine comparison

---

## What are Containers

A **container** is a lightweight, standalone, executable unit of software that packages application code together with all its dependencies — libraries, runtime, system tools, and configuration — so that it runs reliably across different computing environments.

Containers leverage Linux kernel features (namespaces and cgroups) to provide process-level isolation without requiring a full guest operating system, making them far more efficient than traditional virtual machines.

### Traditional Deployments vs VMs vs Containers

```
Traditional Deployment        Virtualized Deployment        Containerized Deployment
┌──────────────────┐         ┌──────────────────┐         ┌──────────────────┐
│   App A   App B  │         │  ┌─────┐ ┌─────┐ │         │ ┌──┐ ┌──┐ ┌──┐  │
│                  │         │  │VM A │ │VM B │ │         │ │C1│ │C2│ │C3│  │
│   Shared Libs    │         │  │App A│ │App B│ │         │ └──┘ └──┘ └──┘  │
│                  │         │  │Bins │ │Bins │ │         │   Container      │
│                  │         │  │Guest│ │Guest│ │         │    Runtime       │
│   Host OS        │         │  │ OS  │ │ OS  │ │         │  (containerd)   │
│   Hardware       │         │  └─────┘ └─────┘ │         │                  │
└──────────────────┘         │   Hypervisor      │         │   Host OS        │
                             │   Host OS         │         │   Hardware       │
                             │   Hardware        │         └──────────────────┘
                             └──────────────────┘
 ● No isolation              ● Strong isolation             ● Process isolation
 ● Dependency conflicts      ● Heavy (GB-sized images)      ● Lightweight (MB-sized)
 ● Hard to reproduce         ● Slow boot (minutes)          ● Fast boot (seconds)
 ● Works-on-my-machine       ● Resource overhead            ● Near-native performance
```

### OCI and the Standards Behind Containers

The **Open Container Initiative (OCI)**, established in 2015 under the Linux Foundation, defines industry standards for container image formats and runtimes. This ensures containers built with any OCI-compliant tool can run on any OCI-compliant runtime.

### Benefits of Containerization

| Benefit | Description |
|---|---|
| **Portability** | Build once, run anywhere — laptop, CI server, staging, production, any cloud |
| **Consistency** | Identical environment from development through production eliminates "works on my machine" |
| **Isolation** | Each container runs in its own namespace with its own filesystem, network, and process tree |
| **Efficiency** | Containers share the host kernel; no guest OS overhead means higher density per host |
| **Speed** | Containers start in milliseconds to seconds, compared to minutes for VMs |
| **Immutability** | Container images are read-only layers; deploy a new version by replacing the container |
| **Scalability** | Lightweight footprint enables rapid horizontal scaling |
| **DevOps enablement** | Standard packaging format bridges the gap between development and operations |

---

## Docker Architecture

Docker uses a **client-server architecture**. The Docker client talks to the Docker daemon, which does the heavy lifting of building, running, and distributing containers. Under the hood, the daemon delegates container execution to **containerd** and **runc**.

```
                         Docker Architecture
┌──────────────────────────────────────────────────────────────────┐
│                        Docker Client                             │
│                                                                  │
│  $ docker build    $ docker run    $ docker pull    $ docker ps  │
│                                                                  │
└──────────────────────┬───────────────────────────────────────────┘
                       │  REST API (unix socket / TCP)
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                     Docker Daemon (dockerd)                       │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │ Image Mgmt   │  │ Network Mgmt │  │  Volume Mgmt           │ │
│  │              │  │              │  │                        │ │
│  │ Build/Pull/  │  │ bridge, host │  │ Named volumes, bind    │ │
│  │ Push/Tag     │  │ overlay, none│  │ mounts, tmpfs          │ │
│  └──────┬───────┘  └──────────────┘  └────────────────────────┘ │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  containerd                               │   │
│  │                                                           │   │
│  │  Container lifecycle management, image transfer,          │   │
│  │  storage management, network attachment                   │   │
│  │                                                           │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │                   containerd-shim                    │ │   │
│  │  │  (per-container process — allows daemon restarts     │ │   │
│  │  │   without killing running containers)                │ │   │
│  │  └──────────────────────┬──────────────────────────────┘ │   │
│  │                         │                                 │   │
│  │                         ▼                                 │   │
│  │              ┌──────────────────┐                         │   │
│  │              │      runc        │                         │   │
│  │              │                  │                         │   │
│  │              │ OCI-compliant    │                         │   │
│  │              │ container runtime│                         │   │
│  │              │ (spawns the      │                         │   │
│  │              │  container       │                         │   │
│  │              │  process)        │                         │   │
│  │              └──────────────────┘                         │   │
│  └───────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### Docker Client

The Docker client (`docker`) is the primary interface for interacting with Docker. When you run a command like `docker run`, the client sends the request to the Docker daemon via a REST API over a Unix socket (`/var/run/docker.sock`) or TCP.

### Docker Daemon (dockerd)

The Docker daemon listens for Docker API requests and manages Docker objects — images, containers, networks, and volumes. The daemon can communicate with other daemons to manage distributed services.

### containerd

**containerd** is a CNCF Graduated container runtime that manages the complete container lifecycle on a host: image pull and push, container execution and supervision, storage, and networking. Docker delegates all container operations to containerd.

### runc

**runc** is a lightweight, OCI-compliant container runtime that creates and runs containers. It is the reference implementation of the OCI Runtime Specification. containerd uses runc to spawn container processes.

### Docker Desktop vs Docker Engine

Docker is available in two primary forms:

- **Docker Engine** — the open-source daemon, CLI, and API for Linux servers
- **Docker Desktop** — a desktop application for macOS, Windows, and Linux that bundles Docker Engine, a Linux VM (on macOS/Windows), Docker Compose, Docker Build, and a GUI

---

## Container Fundamentals

Containers are not a single kernel feature but a combination of several Linux kernel primitives working together: **namespaces** for isolation, **cgroups** for resource limits, and **union filesystems** for efficient image layering.

### Namespaces

Linux namespaces provide the isolation that makes a process inside a container appear to have its own system. Each namespace type isolates a different system resource.

| Namespace | Flag | What It Isolates |
|---|---|---|
| **PID** | `CLONE_NEWPID` | Process IDs — container sees its own PID 1 |
| **NET** | `CLONE_NEWNET` | Network stack — own interfaces, IP addresses, routing tables, ports |
| **MNT** | `CLONE_NEWNS` | Mount points — container has its own filesystem hierarchy |
| **UTS** | `CLONE_NEWUTS` | Hostname and domain name — container can set its own hostname |
| **IPC** | `CLONE_NEWIPC` | Inter-process communication — own message queues, semaphores, shared memory |
| **USER** | `CLONE_NEWUSER` | User and group IDs — root inside container maps to unprivileged user outside |

```bash
# View the namespaces of a running container
docker inspect --format '{{.State.Pid}}' my-container
# Then examine its namespaces in /proc
ls -la /proc/<PID>/ns/

# Example output:
# lrwxrwxrwx 1 root root 0 Jan 15 10:00 cgroup -> cgroup:[4026531835]
# lrwxrwxrwx 1 root root 0 Jan 15 10:00 ipc -> ipc:[4026532389]
# lrwxrwxrwx 1 root root 0 Jan 15 10:00 mnt -> mnt:[4026532387]
# lrwxrwxrwx 1 root root 0 Jan 15 10:00 net -> net:[4026532392]
# lrwxrwxrwx 1 root root 0 Jan 15 10:00 pid -> pid:[4026532390]
# lrwxrwxrwx 1 root root 0 Jan 15 10:00 user -> user:[4026531837]
# lrwxrwxrwx 1 root root 0 Jan 15 10:00 uts -> uts:[4026532388]
```

### Cgroups (Control Groups)

Control groups limit and account for the resource usage (CPU, memory, disk I/O, network) of a collection of processes. Docker uses cgroups to prevent any single container from consuming all host resources.

| Resource | cgroup Controller | Docker Flag | Example |
|---|---|---|---|
| **CPU** | `cpu`, `cpuset` | `--cpus`, `--cpu-shares` | `--cpus="1.5"` limits to 1.5 CPU cores |
| **Memory** | `memory` | `--memory`, `--memory-swap` | `--memory="512m"` limits to 512 MB |
| **I/O** | `blkio` | `--device-read-bps`, `--device-write-bps` | `--device-read-bps /dev/sda:1mb` |
| **PIDs** | `pids` | `--pids-limit` | `--pids-limit=100` limits process count |

```bash
# Run a container with CPU and memory limits
docker run -d \
  --name constrained-app \
  --cpus="1.0" \
  --memory="256m" \
  --memory-swap="512m" \
  --pids-limit=50 \
  nginx:latest

# Verify the limits
docker stats constrained-app --no-stream
```

### Union Filesystems (overlay2)

Docker images are built from **read-only layers** stacked on top of each other using a union filesystem. When a container runs, a thin **writable layer** is added on top. This architecture enables efficient storage and fast image pulls.

```
Container Layer  (read-write)   ← Container writes here
─────────────────────────────
Image Layer 4    (read-only)    ← COPY app.jar /app/
Image Layer 3    (read-only)    ← RUN apt-get install -y openjdk-17
Image Layer 2    (read-only)    ← RUN apt-get update
Image Layer 1    (read-only)    ← FROM ubuntu:22.04 (base image layers)
```

```bash
# Inspect the layers of an image
docker history nginx:latest

# View the storage driver and layer details
docker inspect nginx:latest --format '{{.RootFS.Layers}}'

# Check which storage driver Docker is using
docker info --format '{{.Driver}}'
```

### Container vs Image

| Aspect | Image | Container |
|---|---|---|
| **State** | Immutable, read-only template | Running (or stopped) instance of an image |
| **Analogy** | Class definition | Object instance |
| **Storage** | Read-only layers stacked via union filesystem | Image layers + thin writable layer on top |
| **Creation** | Built with `docker build` from a Dockerfile | Created with `docker create` or `docker run` |
| **Sharing** | Pushed to / pulled from a registry | Exists only on the host where it runs |
| **Persistence** | Persists until explicitly deleted | Writable layer lost when container is removed |

---

## OCI Standards

The **Open Container Initiative (OCI)** defines three specifications that ensure interoperability across the container ecosystem. Any tool that conforms to these specs can build, distribute, and run containers interchangeably.

### Specifications

| Specification | Purpose | Key Defines |
|---|---|---|
| **OCI Runtime Spec** | How to run a container | Container configuration, lifecycle operations (create, start, kill, delete), Linux-specific settings (namespaces, cgroups, mounts) |
| **OCI Image Spec** | How to package a container image | Image manifest, image index (multi-arch), filesystem layers (tar+gzip), image configuration (env, entrypoint, cmd) |
| **OCI Distribution Spec** | How to distribute container images | Registry API for push, pull, and content discovery; tag listing; manifest and blob endpoints |

### OCI-Compliant Tools

```
               BUILD                    DISTRIBUTE                  RUN
  ┌──────────────────────┐    ┌───────────────────────┐    ┌──────────────────┐
  │  docker build        │    │  Docker Hub            │    │  runc            │
  │  buildah             │───►│  GitHub Container Reg  │───►│  crun            │
  │  kaniko              │    │  Amazon ECR            │    │  youki            │
  │  BuildKit            │    │  Azure ACR             │    │  kata-containers │
  │  podman build        │    │  Harbor                │    │  gVisor (runsc)  │
  └──────────────────────┘    │  Quay.io               │    └──────────────────┘
                              └───────────────────────┘
                              All use OCI Distribution Spec
```

---

## Docker CLI Essentials

### Building and Running

```bash
# Build an image from a Dockerfile in the current directory
docker build -t my-app:1.0 .

# Build with a specific Dockerfile and build arguments
docker build -f Dockerfile.prod --build-arg ENV=production -t my-app:1.0 .

# Run a container in the foreground
docker run --name my-app -p 8080:80 my-app:1.0

# Run a container in detached mode with environment variables
docker run -d \
  --name my-app \
  -p 8080:80 \
  -e DATABASE_URL=postgres://db:5432/app \
  -v app-data:/data \
  my-app:1.0

# Run an interactive container (useful for debugging)
docker run -it --rm ubuntu:22.04 /bin/bash
```

### Inspecting and Debugging

```bash
# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# View container logs
docker logs my-app

# Stream logs in real time
docker logs -f --tail 100 my-app

# Execute a command inside a running container
docker exec -it my-app /bin/sh

# Inspect container details (JSON output)
docker inspect my-app

# View resource usage statistics
docker stats

# View port mappings
docker port my-app
```

### Managing Containers

```bash
# Stop a running container (sends SIGTERM, then SIGKILL after timeout)
docker stop my-app

# Start a stopped container
docker start my-app

# Restart a container
docker restart my-app

# Pause all processes in a container (uses cgroup freezer)
docker pause my-app

# Unpause a paused container
docker unpause my-app

# Remove a stopped container
docker rm my-app

# Force remove a running container
docker rm -f my-app

# Remove all stopped containers
docker container prune
```

### Managing Images

```bash
# List local images
docker images

# Pull an image from a registry
docker pull nginx:1.27

# Push an image to a registry
docker tag my-app:1.0 registry.example.com/my-app:1.0
docker push registry.example.com/my-app:1.0

# Remove a local image
docker rmi my-app:1.0

# Remove all unused images
docker image prune -a

# View image layer history
docker history my-app:1.0
```

### System Maintenance

```bash
# Show Docker disk usage
docker system df

# Remove all unused data (containers, images, networks, build cache)
docker system prune -a --volumes

# View Docker daemon information
docker info

# View Docker client and server versions
docker version
```

---

## Container Lifecycle

A Docker container transitions through a well-defined set of states. Understanding this lifecycle is essential for managing and debugging containers.

```
                          Docker Container Lifecycle

     docker create              docker start
  ┌───────────────────┐     ┌───────────────────┐
  │                   │     │                   │
  │                   ▼     │                   ▼
  │             ┌──────────┐│            ┌─────────────┐
  │             │          ││            │             │
  │             │ CREATED  ├┘            │   RUNNING   │◄──────┐
  │             │          │             │             │       │
  │             └──────────┘             └──────┬──────┘       │
  │                                            │              │
  │                            ┌───────────────┼──────────┐   │
  │                            │               │          │   │
  │                            ▼               ▼          │   │
  │                     ┌──────────┐    ┌──────────┐      │   │
  │                     │          │    │          │      │   │
  │                     │  PAUSED  │    │ STOPPED  │      │   │
  │                     │          │    │ (Exited) │      │   │
  │                     └─────┬────┘    └─────┬────┘      │   │
  │                           │               │           │   │
  │               docker      │   docker      │   docker  │   │
  │               unpause     │   start       │   restart │   │
  │                           │               │           │   │
  │                           └───────────────┴───────────┘   │
  │                                           │               │
  │                                           └───────────────┘
  │
  │                        docker rm
  │   ┌──────────────────────────────────────────────────┐
  │   │                                                  │
  │   │           ┌──────────┐                           │
  │   └──────────►│          │◄── (from CREATED          │
  │               │ REMOVED  │     or STOPPED)           │
  └──────────────►│          │                           │
                  └──────────┘                           │
                                                         │
```

### Lifecycle Commands

| Transition | Command | Description |
|---|---|---|
| → Created | `docker create <image>` | Creates a container from an image but does not start it |
| Created → Running | `docker start <container>` | Starts a created or stopped container |
| → Running (shortcut) | `docker run <image>` | Creates and starts a container in one step |
| Running → Paused | `docker pause <container>` | Freezes all processes using the cgroup freezer |
| Paused → Running | `docker unpause <container>` | Resumes a paused container |
| Running → Stopped | `docker stop <container>` | Sends SIGTERM, waits grace period, then SIGKILL |
| Running → Stopped | `docker kill <container>` | Sends SIGKILL immediately |
| Stopped → Running | `docker start <container>` | Restarts a stopped container |
| Running → Restarted | `docker restart <container>` | Stops and then starts the container |
| Stopped → Removed | `docker rm <container>` | Deletes the container and its writable layer |
| Running → Removed | `docker rm -f <container>` | Force removes a running container |

### Restart Policies

Docker supports automatic restart policies to keep containers running after failures or host reboots.

```bash
# Always restart the container unless explicitly stopped
docker run -d --restart unless-stopped --name web nginx:latest

# Restart on failure with a maximum retry count
docker run -d --restart on-failure:5 --name worker my-worker:1.0
```

| Policy | Behavior |
|---|---|
| `no` | Do not restart (default) |
| `on-failure[:max]` | Restart only if the container exits with a non-zero exit code |
| `always` | Always restart regardless of exit code |
| `unless-stopped` | Like `always`, but do not restart if the container was explicitly stopped |

---

## Docker Desktop vs Docker Engine

| Feature | Docker Desktop | Docker Engine |
|---|---|---|
| **Platform** | macOS, Windows, Linux (GUI) | Linux only |
| **License** | Free for personal / education / small business; paid for large enterprises | Free and open-source (Apache 2.0) |
| **Installation** | Graphical installer; includes everything | Package manager (`apt`, `yum`, `dnf`) |
| **Linux VM** | Runs a lightweight Linux VM on macOS/Windows | Runs natively on the Linux kernel |
| **Docker Compose** | Bundled | Install separately as a plugin (`docker-compose-plugin`) |
| **BuildKit** | Enabled by default | Enabled by default (Docker 23.0+) |
| **Kubernetes** | Single-node K8s cluster built-in (toggle on/off) | Not included; install separately |
| **GUI** | Dashboard for containers, images, volumes, extensions | CLI only |
| **Extensions** | Marketplace for third-party tools | Not available |
| **Automatic updates** | Yes | Manual via package manager |
| **Use case** | Developer workstations | Production servers, CI/CD runners |

```bash
# Docker Engine installation on Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Verify installation
docker version
docker run hello-world

# (Optional) Run Docker as non-root
sudo usermod -aG docker $USER
newgrp docker
```

---

## Prerequisites

### Required Knowledge

Before working through the Docker & Containers series, you should be familiar with:

| Topic | Why It Matters |
|---|---|
| **Linux command line** | Docker runs on Linux; navigating the shell, managing files, and inspecting processes is essential |
| **Basic networking** | Containers expose ports, communicate over bridges, and resolve DNS — understanding TCP/IP helps |
| **Package management** | Installing Docker, runtime dependencies, and debugging library issues |
| **Version control (Git)** | Dockerfiles and Compose files should be versioned alongside application code |
| **A programming language** | Building application images requires understanding your app's build and runtime requirements |

### Required Tools

Install the following tools to follow hands-on examples:

```bash
# Docker Engine (Ubuntu/Debian)
sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Docker Engine (macOS via Homebrew — installs Docker Desktop)
brew install --cask docker

# Docker Compose plugin
sudo apt-get install -y docker-compose-plugin
# or standalone
pip install docker-compose

# Verify Docker installation
docker version
docker compose version

# jq (for parsing JSON output from docker inspect)
# macOS
brew install jq
# Ubuntu/Debian
sudo apt-get install -y jq

# curl (for testing containerized services)
# Usually pre-installed; if not:
sudo apt-get install -y curl

# dive (inspect image layers and optimize size)
# macOS
brew install dive
# Linux
wget https://github.com/wagoodman/dive/releases/latest/download/dive_linux_amd64.deb
sudo dpkg -i dive_linux_amd64.deb

# hadolint (Dockerfile linter)
# macOS
brew install hadolint
# Linux
wget -O /usr/local/bin/hadolint \
  https://github.com/hadolint/hadolint/releases/latest/download/hadolint-Linux-x86_64
chmod +x /usr/local/bin/hadolint
```

---

## Next Steps

Work through the Docker & Containers series in order, or jump to the topic most relevant to you:

| File | Topic | Description |
|---|---|---|
| [01-IMAGES-AND-DOCKERFILES.md](01-IMAGES-AND-DOCKERFILES.md) | Images & Dockerfiles | Building images, Dockerfile best practices, multi-stage builds, layer caching |
| [02-CONTAINER-RUNTIME.md](02-CONTAINER-RUNTIME.md) | Container Runtime | Docker Engine, containerd, runc, container lifecycle internals |
| [03-NETWORKING.md](03-NETWORKING.md) | Container Networking | Bridge, host, overlay networks, DNS, port mapping, network security |
| [04-STORAGE-AND-VOLUMES.md](04-STORAGE-AND-VOLUMES.md) | Storage & Volumes | Volumes, bind mounts, tmpfs, storage drivers, data persistence |
| [05-DOCKER-COMPOSE.md](05-DOCKER-COMPOSE.md) | Docker Compose | Multi-container apps, compose files, service dependencies, profiles |
| [06-REGISTRIES.md](06-REGISTRIES.md) | Container Registries | Docker Hub, ECR, ACR, GCR, private registries, image distribution |
| [07-SECURITY.md](07-SECURITY.md) | Container Security | Image scanning, rootless containers, secrets management, supply chain security |
| [08-PERFORMANCE.md](08-PERFORMANCE.md) | Performance | Resource limits, cgroups, image layer optimization, build caching |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Production patterns, 12-factor apps, health checks, graceful shutdown |
| [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md) | Anti-Patterns | Common container mistakes and how to avoid them |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Structured learning guide with exercises |

### Suggested Learning Path by Role

```
Developer:
  00-OVERVIEW → 01-IMAGES-AND-DOCKERFILES → 05-DOCKER-COMPOSE → 08-PERFORMANCE

DevOps / Platform Engineer:
  00-OVERVIEW → 01-IMAGES-AND-DOCKERFILES → 03-NETWORKING → 04-STORAGE-AND-VOLUMES → 05-DOCKER-COMPOSE

SRE:
  00-OVERVIEW → 07-SECURITY → 08-PERFORMANCE → 03-NETWORKING → 09-BEST-PRACTICES

Architect:
  00-OVERVIEW → 02-CONTAINER-RUNTIME → 07-SECURITY → 06-REGISTRIES → 10-ANTI-PATTERNS
```

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Docker & Containers overview documentation |
