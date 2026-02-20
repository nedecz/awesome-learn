# Docker Performance

A comprehensive guide to optimizing Docker container performance — covering resource limits with cgroups, CPU and memory management, I/O tuning, image layer optimization, build performance, startup optimization, monitoring, storage drivers, and networking performance.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Resource Limits with Cgroups](#resource-limits-with-cgroups)
   - [How Cgroups Work](#how-cgroups-work)
   - [Cgroups v1 vs v2](#cgroups-v1-vs-v2)
3. [CPU Management](#cpu-management)
   - [CPU Shares](#cpu-shares)
   - [CPU Quota and Period](#cpu-quota-and-period)
   - [CPU Pinning](#cpu-pinning)
   - [CPU Management Comparison](#cpu-management-comparison)
4. [Memory Management](#memory-management)
   - [Hard Memory Limits](#hard-memory-limits)
   - [Memory Reservation](#memory-reservation)
   - [Swap Configuration](#swap-configuration)
   - [OOM Killer Behavior](#oom-killer-behavior)
5. [I/O Management](#io-management)
   - [Block I/O Weight](#block-io-weight)
   - [Device Read/Write Limits](#device-readwrite-limits)
6. [PID Limits](#pid-limits)
7. [Image Layer Optimization](#image-layer-optimization)
   - [Reducing Image Size](#reducing-image-size)
   - [Layer Caching](#layer-caching)
   - [Dockerignore Effectiveness](#dockerignore-effectiveness)
   - [Image Size Comparison](#image-size-comparison)
8. [Build Performance](#build-performance)
   - [BuildKit Parallelism](#buildkit-parallelism)
   - [Cache Mounts](#cache-mounts)
   - [Inline Cache](#inline-cache)
   - [Registry Cache](#registry-cache)
9. [Container Startup Optimization](#container-startup-optimization)
   - [Init Systems](#init-systems)
   - [Pre-Warming](#pre-warming)
   - [Lazy Pulling](#lazy-pulling)
10. [Monitoring Container Performance](#monitoring-container-performance)
    - [docker stats](#docker-stats)
    - [cAdvisor](#cadvisor)
    - [Prometheus Metrics](#prometheus-metrics)
11. [Storage Driver Performance](#storage-driver-performance)
    - [Storage Driver Comparison](#storage-driver-comparison)
    - [Choosing a Storage Driver](#choosing-a-storage-driver)
12. [Networking Performance](#networking-performance)
    - [Network Driver Benchmarks](#network-driver-benchmarks)
    - [DNS Caching](#dns-caching)
    - [Connection Pooling](#connection-pooling)
13. [Performance Profiling Tools](#performance-profiling-tools)
    - [docker stats](#docker-stats-1)
    - [ctop](#ctop)
    - [cAdvisor Dashboard](#cadvisor-dashboard)
    - [Prometheus cAdvisor Exporter](#prometheus-cadvisor-exporter)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

Container performance is not automatic — Docker provides powerful resource controls through Linux cgroups, but they must be explicitly configured. Without resource limits, a single container can starve an entire host of CPU, memory, or I/O bandwidth. Without build optimization, CI/CD pipelines waste minutes on avoidable cache misses and oversized images.

This document covers the full spectrum of Docker performance: how to limit and manage resources, how to build efficient images, how to reduce startup times, and how to monitor and profile container workloads.

### Target Audience

- **DevOps Engineers** configuring container resource limits and build pipelines
- **Developers** optimizing Dockerfiles and image sizes
- **SREs** monitoring and troubleshooting container performance in production
- **Platform Engineers** selecting storage drivers and tuning infrastructure

### Scope

- Resource management with cgroups (CPU, memory, I/O, PIDs)
- Image layer optimization and build performance
- Container startup and runtime optimization
- Monitoring and profiling tools
- Storage driver and network driver performance

---

## Resource Limits with Cgroups

### How Cgroups Work

Docker uses Linux **control groups (cgroups)** to limit, account for, and isolate resource usage. Every container gets its own cgroup that tracks and constrains CPU, memory, I/O, and process count.

```
Cgroup Resource Control
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────────────────────────────────────────┐
  │                  Host System                      │
  │                                                  │
  │   Total: 8 CPUs, 32 GB RAM, 1 Gbps disk I/O     │
  │                                                  │
  │  ┌──────────────┐  ┌──────────────┐              │
  │  │ Container A  │  │ Container B  │              │
  │  │              │  │              │              │
  │  │ CPU: 2 cores │  │ CPU: 4 cores │              │
  │  │ RAM: 4 GB    │  │ RAM: 8 GB    │              │
  │  │ I/O: 200 MB/s│  │ I/O: 500 MB/s│              │
  │  │ PIDs: 100    │  │ PIDs: 200    │              │
  │  │              │  │              │              │
  │  │  (cgroup A)  │  │  (cgroup B)  │              │
  │  └──────────────┘  └──────────────┘              │
  │                                                  │
  │  Unallocated: 2 CPUs, 20 GB RAM                  │
  └──────────────────────────────────────────────────┘
```

### Cgroups v1 vs v2

| Feature | Cgroups v1 | Cgroups v2 |
|---|---|---|
| **Hierarchy** | Multiple hierarchies (one per controller) | Unified single hierarchy |
| **Resource control** | Per-controller configuration | Unified resource management |
| **Memory + I/O** | Cannot combine memory and I/O limits | ✅ Unified memory + I/O pressure |
| **PSI (Pressure Stall Info)** | ❌ | ✅ CPU, memory, I/O pressure metrics |
| **Default in Docker** | Docker 20.10 and earlier | Docker 23+ on modern kernels |

```bash
# Check which cgroup version is active
stat -fc %T /sys/fs/cgroup/
# tmpfs    = cgroups v1
# cgroup2fs = cgroups v2

# View a container's cgroup
docker inspect --format '{{.HostConfig.CgroupParent}}' <container>
cat /sys/fs/cgroup/docker/<container-id>/memory.max
```

---

## CPU Management

### CPU Shares

CPU shares (`--cpu-shares`) set relative CPU weight. The default is 1024. This only matters when CPU is contested — if the host has idle CPU, any container can use 100%.

```bash
# Default shares (1024)
docker run -d --name app myapp:latest

# Double share weight — gets 2x CPU time under contention
docker run -d --cpu-shares 2048 --name priority-app myapp:latest

# Half share weight — gets 0.5x CPU time under contention
docker run -d --cpu-shares 512 --name background-job myapp:latest
```

```
CPU Share Allocation Under Contention
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Total: 4 CPU cores (fully contested)

  Container A (shares=2048):  2048/3584 = 57% → 2.28 cores
  Container B (shares=1024):  1024/3584 = 29% → 1.14 cores
  Container C (shares=512):    512/3584 = 14% → 0.57 cores
                               ──────
                               3584 total shares
```

### CPU Quota and Period

The `--cpus` flag sets a hard limit on CPU usage regardless of contention.

```bash
# Limit to 1.5 CPU cores
docker run -d --cpus=1.5 myapp:latest

# Equivalent using --cpu-quota and --cpu-period
# quota/period = 150000/100000 = 1.5 CPUs
docker run -d --cpu-quota=150000 --cpu-period=100000 myapp:latest

# Limit to 0.5 CPU cores (half a core)
docker run -d --cpus=0.5 myapp:latest

# Limit to exactly 2 cores
docker run -d --cpus=2 myapp:latest
```

| Flag | Description | Example | Effect |
|---|---|---|---|
| `--cpus` | Hard CPU limit (decimal) | `--cpus=1.5` | Max 1.5 cores |
| `--cpu-shares` | Relative weight (contested only) | `--cpu-shares=2048` | 2x priority |
| `--cpu-quota` | Microseconds of CPU per period | `--cpu-quota=150000` | 150ms per 100ms period |
| `--cpu-period` | CFS period in microseconds | `--cpu-period=100000` | 100ms (default) |

### CPU Pinning

Pin containers to specific CPU cores to optimize cache locality and reduce context switching.

```bash
# Pin to CPU cores 0 and 1
docker run -d --cpuset-cpus="0,1" myapp:latest

# Pin to cores 0 through 3
docker run -d --cpuset-cpus="0-3" myapp:latest

# Pin to specific NUMA node CPUs (cores 0-7 on node 0)
docker run -d --cpuset-cpus="0-7" --cpuset-mems="0" myapp:latest
```

> **When to pin CPUs:** CPU pinning is useful for latency-sensitive workloads (databases, real-time processing) where cache locality matters. Avoid pinning for general web applications — the scheduler does a better job distributing load.

### CPU Management Comparison

| Strategy | When to Use | Example |
|---|---|---|
| **No limits** | Development only | `docker run myapp` |
| **Shares only** | Fair sharing, burst allowed | `--cpu-shares=2048` |
| **Hard limit** | Production, predictable capacity | `--cpus=2` |
| **Pinning + limit** | Latency-sensitive, NUMA-aware | `--cpuset-cpus="0,1" --cpus=2` |

---

## Memory Management

### Hard Memory Limits

The `--memory` flag sets a hard upper limit. The container is killed (OOM) if it exceeds this limit.

```bash
# Limit container to 512 MB
docker run -d --memory=512m myapp:latest

# Limit to 2 GB
docker run -d --memory=2g myapp:latest

# Limit memory and disable swap
docker run -d --memory=1g --memory-swap=1g myapp:latest
```

### Memory Reservation

Memory reservation (`--memory-reservation`) is a soft limit — Docker tries to keep the container under this limit but allows bursting up to `--memory`.

```bash
# Soft limit 256 MB, hard limit 512 MB
docker run -d \
  --memory=512m \
  --memory-reservation=256m \
  myapp:latest
```

```
Memory Limit Behavior
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  512 MB ┬─────────────────── Hard limit (--memory)
         │                    Container killed (OOM) if exceeded
         │
  256 MB ┼─────────────────── Soft limit (--memory-reservation)
         │                    Docker tries to reclaim memory
         │
    0 MB ┴───────────────────
```

### Swap Configuration

```bash
# Allow 1 GB memory + 1 GB swap (total 2 GB)
docker run -d --memory=1g --memory-swap=2g myapp:latest

# Disable swap entirely (recommended for production)
docker run -d --memory=1g --memory-swap=1g myapp:latest

# Unlimited swap (not recommended)
docker run -d --memory=1g --memory-swap=-1 myapp:latest

# Set swap priority (swappiness 0-100, lower = less swap)
docker run -d --memory=1g --memory-swappiness=10 myapp:latest
```

| Flag | Description | Recommendation |
|---|---|---|
| `--memory` | Hard memory limit | Always set in production |
| `--memory-swap` | Memory + swap limit | Set equal to `--memory` to disable swap |
| `--memory-reservation` | Soft limit for scheduler | Set to ~50-75% of `--memory` |
| `--memory-swappiness` | Swap tendency (0-100) | Set low (0-10) for latency-sensitive apps |

### OOM Killer Behavior

When a container exceeds its memory limit, the Linux OOM killer terminates the container's main process.

```bash
# Disable OOM killer (container hangs instead of being killed)
# NOT recommended — use only for debugging
docker run -d --memory=512m --oom-kill-disable myapp:latest

# Adjust OOM score (lower = less likely to be killed)
# Range: -1000 to 1000
docker run -d --memory=1g --oom-score-adj=-500 myapp:latest
```

```bash
# Check if a container was OOM-killed
docker inspect --format '{{.State.OOMKilled}}' <container>

# Check container memory usage
docker stats --no-stream <container>

# View OOM events in system logs
dmesg | grep -i "oom\|killed"
```

> **Production recommendation:** Always set `--memory` limits. Monitor containers approaching their limits with Prometheus and alert at 80% utilization. Never use `--oom-kill-disable` in production.

---

## I/O Management

### Block I/O Weight

Control relative I/O priority between containers when disk bandwidth is contested.

```bash
# Set I/O weight (10-1000, default 500)
docker run -d --blkio-weight=100 --name background myapp:latest
docker run -d --blkio-weight=900 --name critical myapp:latest

# Set per-device weight
docker run -d --blkio-weight-device="/dev/sda:200" myapp:latest
```

### Device Read/Write Limits

Set absolute limits on device I/O bandwidth.

```bash
# Limit read speed to 10 MB/s
docker run -d --device-read-bps=/dev/sda:10mb myapp:latest

# Limit write speed to 5 MB/s
docker run -d --device-write-bps=/dev/sda:5mb myapp:latest

# Limit read IOPS to 1000
docker run -d --device-read-iops=/dev/sda:1000 myapp:latest

# Limit write IOPS to 500
docker run -d --device-write-iops=/dev/sda:500 myapp:latest

# Combined I/O limits
docker run -d \
  --device-read-bps=/dev/sda:50mb \
  --device-write-bps=/dev/sda:30mb \
  --device-read-iops=/dev/sda:2000 \
  --device-write-iops=/dev/sda:1000 \
  myapp:latest
```

| Flag | Description | Use Case |
|---|---|---|
| `--blkio-weight` | Relative I/O weight (10-1000) | Fair sharing between containers |
| `--device-read-bps` | Read bandwidth limit | Prevent I/O-heavy containers from starving others |
| `--device-write-bps` | Write bandwidth limit | Protect disk from log storms |
| `--device-read-iops` | Read IOPS limit | Database workload management |
| `--device-write-iops` | Write IOPS limit | Limit random write pressure |

---

## PID Limits

Limit the number of processes a container can create to prevent fork bombs and resource exhaustion.

```bash
# Limit to 100 processes
docker run -d --pids-limit=100 myapp:latest

# Set a global default in daemon.json
# /etc/docker/daemon.json
# { "default-pids-limit": 200 }
```

---

## Image Layer Optimization

### Reducing Image Size

Every megabyte in your image multiplies across registries, networks, and nodes. Smaller images pull faster, start faster, and have a smaller attack surface.

```dockerfile
# ❌ Bad: 1.2 GB image
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]

# ✅ Good: 150 MB image (multi-stage, alpine, production deps)
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./
USER node
CMD ["node", "dist/server.js"]
```

**Image size reduction techniques:**

| Technique | Typical Savings | Example |
|---|---|---|
| **Alpine base** | 60-90% | `node:20` (1 GB) → `node:20-alpine` (180 MB) |
| **Multi-stage builds** | 50-95% | Copy only build artifacts to final stage |
| **Combine RUN commands** | 10-30% | Single RUN with `&&` removes intermediate layers |
| **Remove package cache** | 5-20% | `rm -rf /var/lib/apt/lists/*` |
| **.dockerignore** | Variable | Exclude `node_modules`, `.git`, test files |
| **Distroless / scratch** | 90-99% | For statically compiled binaries |

### Layer Caching

Docker caches each layer. When a layer changes, all subsequent layers are rebuilt. Order instructions from least to most frequently changing.

```dockerfile
# ✅ Optimized for cache hits
FROM python:3.12-slim

# Layer 1: System deps (rarely change)
RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# Layer 2: Python deps (change occasionally)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Layer 3: Application code (changes frequently)
COPY . .

USER nobody
CMD ["gunicorn", "app:app"]
```

```
Layer Cache Behavior
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  FROM python:3.12-slim       ← cached ✅
  RUN apt-get install...      ← cached ✅
  COPY requirements.txt       ← cached ✅ (if unchanged)
  RUN pip install...          ← cached ✅ (if requirements unchanged)
  COPY . .                    ← REBUILT (code changed)
  CMD ["gunicorn"...]         ← REBUILT (layer above changed)
```

### Dockerignore Effectiveness

A well-crafted `.dockerignore` prevents unnecessary files from entering the build context, reducing build time and image size.

```
# .dockerignore
.git
.github
.gitignore
node_modules
npm-debug.log
Dockerfile
docker-compose*.yml
.dockerignore
.env
.env.*
*.md
!README.md
tests/
coverage/
.nyc_output/
.vscode/
.idea/
__pycache__/
*.pyc
.pytest_cache/
dist/
build/
```

```bash
# Check build context size
docker build --no-cache -t test . 2>&1 | grep "Sending build context"
# Sending build context to Docker daemon  2.048kB  ← good
# Sending build context to Docker daemon  450.3MB  ← too large, check .dockerignore
```

### Image Size Comparison

| Language | Full Image | Alpine | Distroless | Scratch |
|---|---|---|---|---|
| **Node.js** | 1.1 GB | 180 MB | 130 MB | N/A |
| **Python** | 920 MB | 60 MB | 52 MB | N/A |
| **Go** | 850 MB | 15 MB | 8 MB | 5 MB |
| **Java** | 680 MB | 200 MB | 150 MB | N/A |
| **Rust** | 1.4 GB | 12 MB | 8 MB | 4 MB |

---

## Build Performance

### BuildKit Parallelism

BuildKit (Docker's modern build backend) parallelizes independent build stages automatically.

```dockerfile
# BuildKit builds these stages in parallel
FROM golang:1.23 AS backend
WORKDIR /app
COPY backend/ .
RUN go build -o /server .

FROM node:20-alpine AS frontend
WORKDIR /app
COPY frontend/ .
RUN npm ci && npm run build

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=backend /server /server
COPY --from=frontend /app/dist /static
ENTRYPOINT ["/server"]
```

```bash
# Enable BuildKit (default in Docker 23+)
export DOCKER_BUILDKIT=1

# Build with progress output
docker build --progress=plain -t myapp .

# Build with maximum parallelism
docker buildx build --allow-security.insecure --load -t myapp .
```

### Cache Mounts

Cache mounts persist package manager caches between builds, dramatically speeding up dependency installation.

```dockerfile
# syntax=docker/dockerfile:1

# Go module cache
FROM golang:1.23
WORKDIR /app
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /server .

# Python pip cache
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
COPY . .

# Node.js npm cache
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci
COPY . .

# apt-get cache
FROM debian:bookworm-slim
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y curl
```

### Inline Cache

Export build cache metadata inline with the image for remote cache reuse.

```bash
# Build and push with inline cache metadata
docker buildx build \
  --cache-to type=inline \
  --push \
  -t myregistry.io/myapp:latest .

# Build using the remote image as cache source
docker buildx build \
  --cache-from type=registry,ref=myregistry.io/myapp:latest \
  -t myregistry.io/myapp:v1.0.0 .
```

### Registry Cache

Store build cache separately in a registry for fine-grained cache control.

```bash
# Push cache to a dedicated cache repository
docker buildx build \
  --cache-to type=registry,ref=myregistry.io/myapp:buildcache,mode=max \
  --cache-from type=registry,ref=myregistry.io/myapp:buildcache \
  --push \
  -t myregistry.io/myapp:v1.0.0 .

# Local directory cache (CI runners with persistent storage)
docker buildx build \
  --cache-to type=local,dest=/tmp/buildcache \
  --cache-from type=local,src=/tmp/buildcache \
  -t myapp:latest .
```

| Cache Type | Pros | Cons |
|---|---|---|
| **Inline** | No extra storage; metadata in image | Limited cache data; max mode not supported |
| **Registry** | Full cache; works across machines | Extra registry storage; network transfer |
| **Local** | Fast; no network | Not shared between machines |
| **GitHub Actions** | Integrated with GHA; shared across runs | Size limits (10 GB) |

---

## Container Startup Optimization

### Init Systems

Use an init process (PID 1) to properly handle signals and reap zombie processes.

```bash
# Use Docker's built-in init (tini)
docker run --init myapp:latest

# Use tini directly in Dockerfile
FROM alpine:3.19
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["./myapp"]
```

### Pre-Warming

Pre-warm application caches and connection pools before accepting traffic.

```dockerfile
# Healthcheck that validates warm-up is complete
HEALTHCHECK --interval=5s --timeout=3s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8080/readyz || exit 1
```

```go
// Pre-warm connection pool before accepting traffic
func main() {
    db := connectDB()
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(10)

    // Warm the pool
    for i := 0; i < 10; i++ {
        if err := db.Ping(); err != nil {
            log.Fatal(err)
        }
    }

    // Now start accepting traffic
    http.ListenAndServe(":8080", handler)
}
```

### Lazy Pulling

Lazy pulling (eStargz, Nydus) starts the container before the entire image is downloaded. Only the required layers are fetched on demand.

```bash
# Convert image to eStargz format for lazy pulling
ctr-remote image optimize myregistry.io/myapp:latest myregistry.io/myapp:esgz

# Run with lazy pulling (containerd + stargz-snapshotter)
ctr-remote run --rm myregistry.io/myapp:esgz mycontainer
```

---

## Monitoring Container Performance

### docker stats

The built-in `docker stats` command provides real-time resource usage.

```bash
# Real-time stats for all containers
docker stats

# Stats for specific containers
docker stats web api db

# One-shot (non-streaming) with formatting
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"
```

```
CONTAINER   CPU %   MEM USAGE / LIMIT   NET I/O          BLOCK I/O
web         2.34%   128MiB / 512MiB     1.2MB / 450kB    12MB / 0B
api         15.67%  256MiB / 1GiB       5.6MB / 2.1MB    45MB / 8MB
db          8.12%   2GiB / 4GiB         890kB / 1.5MB    1.2GB / 500MB
```

### cAdvisor

cAdvisor (Container Advisor) provides detailed resource usage and performance metrics.

```bash
# Run cAdvisor as a container
docker run -d \
  --name=cadvisor \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  gcr.io/cadvisor/cadvisor:v0.49.1
```

### Prometheus Metrics

```yaml
# docker-compose.yml — monitoring stack
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"

  prometheus:
    image: prom/prometheus:v2.50.1
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: cadvisor
    static_configs:
      - targets: ["cadvisor:8080"]

  - job_name: docker
    static_configs:
      - targets: ["host.docker.internal:9323"]
```

**Key Prometheus metrics from cAdvisor:**

| Metric | Description | Alert Threshold |
|---|---|---|
| `container_cpu_usage_seconds_total` | Cumulative CPU time consumed | Rate > 90% of limit |
| `container_memory_usage_bytes` | Current memory usage | > 80% of limit |
| `container_memory_working_set_bytes` | Active memory (excludes cache) | > 85% of limit |
| `container_fs_reads_bytes_total` | Cumulative disk read bytes | Sudden spikes |
| `container_fs_writes_bytes_total` | Cumulative disk write bytes | Sustained high rate |
| `container_network_receive_bytes_total` | Network bytes received | > baseline × 3 |
| `container_network_transmit_bytes_total` | Network bytes transmitted | > baseline × 3 |
| `container_oom_events_total` | OOM kill count | > 0 |

---

## Storage Driver Performance

### Storage Driver Comparison

| Storage Driver | Performance | Stability | Copy-on-Write | Best For |
|---|---|---|---|---|
| **overlay2** | ✅ Excellent | ✅ Production-ready | File-level CoW | Default; recommended for most workloads |
| **fuse-overlayfs** | 🟡 Good | ✅ Stable | File-level CoW | Rootless Docker |
| **btrfs** | 🟡 Good | ✅ Stable | Block-level CoW | Btrfs filesystems |
| **zfs** | 🟡 Good | ✅ Stable | Block-level CoW | ZFS filesystems, snapshots |
| **vfs** | ❌ Poor | ✅ Stable | No CoW (full copy) | Testing only; no kernel requirements |
| **devicemapper** | 🟡 Fair | ⚠️ Deprecated | Block-level CoW | Legacy; do not use for new deployments |

### Choosing a Storage Driver

```bash
# Check current storage driver
docker info | grep "Storage Driver"

# Configure storage driver in daemon.json
# /etc/docker/daemon.json
# {
#   "storage-driver": "overlay2",
#   "storage-opts": [
#     "overlay2.override_kernel_check=true"
#   ]
# }

# Check filesystem support
df -T /var/lib/docker
```

```
Storage Driver Decision Tree
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Is this rootless Docker?
  ├── Yes → fuse-overlayfs
  └── No
       │
       Is the filesystem ext4 or xfs?
       ├── Yes → overlay2 ✅ (recommended)
       └── No
            │
            Is it btrfs?
            ├── Yes → btrfs
            └── Is it ZFS?
                 ├── Yes → zfs
                 └── overlay2 (most compatible)
```

---

## Networking Performance

### Network Driver Benchmarks

| Driver | Throughput | Latency Overhead | Use Case |
|---|---|---|---|
| **host** | Native (0% overhead) | 0 μs | Maximum performance, no isolation |
| **bridge** | ~95% of host | ~50 μs | Default; single-host containers |
| **macvlan** | ~98% of host | ~10 μs | Direct LAN attachment |
| **overlay** | ~80-90% of host | ~200-500 μs | Multi-host (Swarm) |
| **none** | N/A | N/A | No networking |

```bash
# Benchmark with iperf3 — host networking
docker run -d --network host --name server networkstatic/iperf3 -s
docker run --network host --rm networkstatic/iperf3 -c 127.0.0.1

# Benchmark with iperf3 — bridge networking
docker run -d --name server networkstatic/iperf3 -s
docker run --rm --link server networkstatic/iperf3 -c server
```

### DNS Caching

Docker's embedded DNS can become a bottleneck under high request rates. Add a local DNS cache.

```yaml
# docker-compose.yml with DNS caching
services:
  dns-cache:
    image: coredns/coredns:1.11
    volumes:
      - ./Corefile:/root/Corefile:ro
    networks:
      - internal

  app:
    image: myapp:latest
    dns:
      - dns-cache
    networks:
      - internal

networks:
  internal:
```

### Connection Pooling

Establish connection pools inside containers to avoid TCP handshake overhead.

```yaml
# docker-compose.yml — PgBouncer connection pooler
services:
  pgbouncer:
    image: edoburu/pgbouncer:1.22
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/mydb
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 200
      DEFAULT_POOL_SIZE: 25
    networks:
      - backend

  app:
    image: myapp:latest
    environment:
      DATABASE_URL: postgres://user:pass@pgbouncer:6432/mydb
    networks:
      - backend

  db:
    image: postgres:16
    networks:
      - backend

networks:
  backend:
```

---

## Performance Profiling Tools

### docker stats

```bash
# Continuous monitoring
docker stats

# Formatted output
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemPerc}}\t{{.MemUsage}}\t{{.PIDs}}"

# JSON output for scripting
docker stats --no-stream --format '{{json .}}' | jq '.'
```

### ctop

ctop provides a `top`-like interface for container metrics.

```bash
# Install ctop
sudo wget https://github.com/bcicen/ctop/releases/download/v0.7.7/ctop-0.7.7-linux-amd64 \
  -O /usr/local/bin/ctop
sudo chmod +x /usr/local/bin/ctop

# Run ctop
ctop

# Filter to specific containers
ctop -f web
```

### cAdvisor Dashboard

cAdvisor provides a web UI at `http://localhost:8080` with per-container dashboards showing CPU, memory, network, and filesystem usage over time.

### Prometheus cAdvisor Exporter

```yaml
# Grafana dashboard queries for container performance

# CPU usage percentage per container
# rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100

# Memory usage percentage per container
# container_memory_working_set_bytes{name!=""} /
# container_spec_memory_limit_bytes{name!=""} * 100

# Network throughput per container
# rate(container_network_receive_bytes_total{name!=""}[5m])

# Disk I/O per container
# rate(container_fs_writes_bytes_total{name!=""}[5m])
```

```
Performance Monitoring Stack
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌────────────────┐    ┌──────────┐    ┌──────────┐
  │   Containers   │───▶│ cAdvisor │───▶│Prometheus│
  │   (metrics)    │    │ (collect)│    │ (store)  │
  └────────────────┘    └──────────┘    └────┬─────┘
                                             │
  ┌────────────────┐                    ┌────▼─────┐
  │ docker stats   │                    │ Grafana  │
  │ ctop           │                    │(visualize│
  │ (quick look)   │                    │ + alert) │
  └────────────────┘                    └──────────┘
```

---

## Next Steps

Continue your Docker learning journey:

| File | Topic | Description |
|---|---|---|
| [07-SECURITY.md](07-SECURITY.md) | Docker Security | Container isolation, image hardening, runtime security, supply chain |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Docker Best Practices | Production patterns, health checks, logging, CI/CD integration |
| [01-IMAGES-AND-DOCKERFILES.md](01-IMAGES-AND-DOCKERFILES.md) | Images & Dockerfiles | Multi-stage builds, Dockerfile instructions, and image layering |
| [05-DOCKER-COMPOSE.md](05-DOCKER-COMPOSE.md) | Docker Compose | Multi-container applications with Compose files |
| [04-STORAGE-AND-VOLUMES.md](04-STORAGE-AND-VOLUMES.md) | Storage & Volumes | Volumes, bind mounts, tmpfs, and storage drivers |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Docker Performance documentation |
