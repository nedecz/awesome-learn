# Docker Security

A comprehensive guide to securing Docker containers, images, and infrastructure — covering the container threat model, image hardening, runtime security, rootless containers, secrets management, network security, and supply chain integrity.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Container Security Landscape](#container-security-landscape)
   - [Threat Model](#threat-model)
   - [Security Layers](#security-layers)
3. [Container Isolation and Attack Surface](#container-isolation-and-attack-surface)
   - [Linux Kernel Isolation Primitives](#linux-kernel-isolation-primitives)
   - [Container Escape Vectors](#container-escape-vectors)
4. [Image Security](#image-security)
   - [Base Image Selection](#base-image-selection)
   - [Vulnerability Scanning](#vulnerability-scanning)
   - [Image Signing with Cosign](#image-signing-with-cosign)
5. [Dockerfile Security Best Practices](#dockerfile-security-best-practices)
   - [Non-Root USER](#non-root-user)
   - [Minimal Base Images](#minimal-base-images)
   - [No Secrets in Layers](#no-secrets-in-layers)
   - [Pin Versions](#pin-versions)
6. [Runtime Security](#runtime-security)
   - [Read-Only Filesystems](#read-only-filesystems)
   - [Dropping Capabilities](#dropping-capabilities)
   - [Seccomp Profiles](#seccomp-profiles)
   - [AppArmor and SELinux](#apparmor-and-selinux)
7. [Rootless Containers](#rootless-containers)
   - [Docker Rootless Mode](#docker-rootless-mode)
   - [Podman](#podman)
8. [Secrets Management](#secrets-management)
   - [Docker Secrets](#docker-secrets)
   - [Build-Time vs Runtime Secrets](#build-time-vs-runtime-secrets)
   - [External Vault Integration](#external-vault-integration)
9. [Network Security](#network-security)
   - [Network Segmentation](#network-segmentation)
   - [TLS Between Containers](#tls-between-containers)
   - [Docker Socket Protection](#docker-socket-protection)
10. [Supply Chain Security](#supply-chain-security)
    - [SBOM Generation](#sbom-generation)
    - [Image Provenance](#image-provenance)
    - [Sigstore and Cosign Workflows](#sigstore-and-cosign-workflows)
11. [Docker Daemon Security](#docker-daemon-security)
    - [TLS for Remote API](#tls-for-remote-api)
    - [User Namespaces](#user-namespaces)
    - [Authorization Plugins](#authorization-plugins)
12. [Container Scanning in CI/CD](#container-scanning-in-cicd)
    - [GitHub Actions Example](#github-actions-example)
    - [GitLab CI Example](#gitlab-ci-example)
13. [Security Checklist](#security-checklist)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

Container security is not a single tool or configuration — it is a layered discipline that spans the entire lifecycle from image build to production runtime. A single misconfiguration (running as root, exposing the Docker socket, embedding secrets in layers) can compromise an entire host.

This document provides a practical, defense-in-depth approach to Docker security covering images, builds, runtime, networking, secrets, and supply chain integrity.

### Target Audience

- **DevOps Engineers** securing container pipelines and infrastructure
- **Developers** building and shipping container images
- **SREs** hardening production Docker environments
- **Security Engineers** assessing container attack surface and compliance

### Scope

- Container threat model and isolation primitives
- Image vulnerability scanning, signing, and hardening
- Dockerfile security best practices
- Runtime hardening (capabilities, seccomp, AppArmor, read-only FS)
- Rootless container modes
- Secrets management patterns
- Network security and supply chain integrity

---

## Container Security Landscape

### Threat Model

Container security threats span multiple layers of the stack. Understanding the threat model helps prioritize hardening efforts.

```
Container Threat Model
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  SUPPLY CHAIN           BUILD TIME              RUNTIME
  ┌──────────────┐   ┌──────────────┐   ┌──────────────────┐
  │ Malicious    │   │ Secrets in   │   │ Container escape │
  │ base images  │   │ image layers │   │ Privilege escal. │
  │ Typosquatting│   │ Excessive    │   │ Resource abuse   │
  │ Compromised  │   │ permissions  │   │ Network lateral  │
  │ dependencies │   │ Unpatched    │   │ movement         │
  │ Unsigned     │   │ packages     │   │ Data exfiltration│
  │ images       │   │ Root user    │   │ Crypto mining    │
  └──────────────┘   └──────────────┘   └──────────────────┘
        │                   │                     │
        └───────────────────┼─────────────────────┘
                            ▼
                    HOST / KERNEL
              (shared kernel = shared risk)
```

| Threat Category | Examples | Mitigation |
|---|---|---|
| **Vulnerable images** | Unpatched CVEs in base images or dependencies | Scan with Trivy/Grype, use minimal bases, update regularly |
| **Secrets exposure** | API keys baked into image layers | Use build secrets, external vaults, never `COPY .env` |
| **Container escape** | Kernel exploits, misconfigured capabilities | Drop capabilities, use seccomp, keep kernel patched |
| **Privilege escalation** | Running as root, writable host mounts | Non-root USER, read-only FS, no `--privileged` |
| **Supply chain attacks** | Compromised upstream images, dependency confusion | Sign images, verify provenance, use SBOMs |
| **Network attacks** | Lateral movement between containers, ARP spoofing | Network segmentation, encrypt inter-container traffic |

### Security Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    SECURITY LAYERS                           │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Layer 5: Supply Chain                                │  │
│  │  Image signing, SBOM, provenance, trusted registries  │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │  Layer 4: Network                                     │  │
│  │  Segmentation, TLS, firewall rules, no exposed socket │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │  Layer 3: Runtime                                     │  │
│  │  Read-only FS, dropped caps, seccomp, AppArmor       │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │  Layer 2: Image / Build                               │  │
│  │  Minimal base, non-root, no secrets, pinned versions  │  │
│  ├───────────────────────────────────────────────────────┤  │
│  │  Layer 1: Host / Kernel                               │  │
│  │  Patched kernel, user namespaces, rootless mode       │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  Defense in depth — compromise of one layer should not      │
│  compromise the entire system.                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Container Isolation and Attack Surface

### Linux Kernel Isolation Primitives

Containers are isolated using Linux kernel features — they are **not** virtual machines. Understanding these primitives is critical for assessing the real isolation boundary.

| Primitive | Purpose | What It Isolates |
|---|---|---|
| **Namespaces** | Resource visibility isolation | PID, network, mount, UTS, IPC, user, cgroup |
| **Cgroups** | Resource usage limits | CPU, memory, I/O, PIDs |
| **Capabilities** | Fine-grained root privileges | 40+ individual capabilities instead of full root |
| **Seccomp** | System call filtering | Block dangerous syscalls (e.g., `mount`, `reboot`) |
| **AppArmor / SELinux** | Mandatory access control | File paths, network, capabilities per profile |
| **Overlay FS** | Layered filesystem | Copy-on-write isolation between containers |

```
Container Isolation Architecture
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────────────────────────────────────────┐
  │                Container A                        │
  │  ┌────────────┐  ┌──────────┐  ┌──────────────┐  │
  │  │ PID NS     │  │ Net NS   │  │ Mount NS     │  │
  │  │ (PID 1     │  │ (eth0    │  │ (overlayfs   │  │
  │  │  = app)    │  │  = veth) │  │  layers)     │  │
  │  └────────────┘  └──────────┘  └──────────────┘  │
  │  ┌────────────┐  ┌──────────┐  ┌──────────────┐  │
  │  │ Seccomp    │  │ Caps     │  │ AppArmor     │  │
  │  │ (syscall   │  │ (dropped │  │ (MAC         │  │
  │  │  filter)   │  │  privs)  │  │  profile)    │  │
  │  └────────────┘  └──────────┘  └──────────────┘  │
  ├──────────────────────────────────────────────────┤
  │              cgroups (resource limits)            │
  ├──────────────────────────────────────────────────┤
  │              Linux Kernel (shared)                │
  └──────────────────────────────────────────────────┘
```

### Container Escape Vectors

Container escapes are the most critical container security risk. Common vectors include:

| Vector | Description | Mitigation |
|---|---|---|
| **Privileged mode** | `--privileged` disables all isolation | Never use `--privileged` in production |
| **Docker socket mount** | `/var/run/docker.sock` gives full host control | Never mount the Docker socket into containers |
| **Kernel exploits** | Shared kernel means kernel CVEs affect all containers | Keep host kernel patched, use seccomp |
| **Writable sensitive mounts** | Bind mounts to `/etc`, `/proc`, `/sys` | Use read-only mounts, limit bind mount paths |
| **CAP_SYS_ADMIN** | Grants near-root-level access | Drop all capabilities, add only what is needed |
| **`--pid=host`** | Shares host PID namespace | Avoid unless absolutely necessary |

---

## Image Security

### Base Image Selection

The choice of base image directly determines your initial vulnerability surface.

| Base Image | Size | Packages | CVE Surface | Use Case |
|---|---|---|---|---|
| `scratch` | 0 MB | None | Minimal | Statically compiled Go/Rust binaries |
| `distroless` | 2–20 MB | Runtime only | Very low | Java, Python, Node.js apps |
| `alpine` | 5 MB | musl + apk | Low | General purpose, small footprint |
| `ubuntu` | 78 MB | apt + many | Medium | When you need apt packages |
| `debian` | 124 MB | apt + many | Medium–High | Broad compatibility |
| `node:20` | 1 GB+ | Full OS + Node | High | Development only |

```dockerfile
# ✅ Good: Multi-stage with distroless
FROM golang:1.23 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /server .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /server /server
ENTRYPOINT ["/server"]
```

### Vulnerability Scanning

Scan images for known CVEs before pushing to a registry.

**Trivy:**

```bash
# Scan a local image
trivy image myapp:latest

# Scan and fail on HIGH or CRITICAL vulnerabilities
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest

# Scan a Dockerfile (config scan)
trivy config Dockerfile

# Generate SARIF output for CI integration
trivy image --format sarif --output trivy-results.sarif myapp:latest

# Scan with a specific database (air-gapped)
trivy image --skip-db-update --db-repository my-mirror/trivy-db myapp:latest
```

**Grype:**

```bash
# Scan a local image
grype myapp:latest

# Fail on HIGH or CRITICAL
grype myapp:latest --fail-on high

# Output as JSON
grype myapp:latest -o json > grype-results.json
```

**Snyk:**

```bash
# Scan a container image
snyk container test myapp:latest

# Monitor for new vulnerabilities
snyk container monitor myapp:latest

# Scan Dockerfile for misconfigurations
snyk iac test Dockerfile
```

### Image Signing with Cosign

Sign images to verify they have not been tampered with between build and deployment.

```bash
# Generate a key pair
cosign generate-key-pair

# Sign an image
cosign sign --key cosign.key myregistry.io/myapp:v1.0.0

# Verify a signature
cosign verify --key cosign.pub myregistry.io/myapp:v1.0.0

# Keyless signing (Sigstore Fulcio + Rekor)
cosign sign myregistry.io/myapp:v1.0.0

# Verify keyless signature
cosign verify \
  --certificate-identity=ci@myorg.com \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  myregistry.io/myapp:v1.0.0
```

---

## Dockerfile Security Best Practices

### Non-Root USER

Running containers as root is the single most common Docker security mistake. A process running as root inside a container is root on the host (unless user namespaces are enabled).

```dockerfile
# ✅ Good: Create a non-root user and switch to it
FROM node:20-alpine

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
COPY --chown=appuser:appgroup package*.json ./
RUN npm ci --only=production
COPY --chown=appuser:appgroup . .

USER appuser
EXPOSE 3000
CMD ["node", "server.js"]
```

```dockerfile
# ❌ Bad: Running as root (default)
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]
# Container runs as root — any exploit has full container access
```

### Minimal Base Images

Fewer packages mean fewer vulnerabilities and smaller attack surface.

```dockerfile
# ✅ Good: Alpine-based for interpreted languages
FROM python:3.12-alpine
RUN pip install --no-cache-dir gunicorn flask
COPY app.py .
USER nobody
CMD ["gunicorn", "-b", "0.0.0.0:8000", "app:app"]
```

```dockerfile
# ✅ Best: Distroless for compiled languages
FROM rust:1.77 AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM gcr.io/distroless/cc-debian12:nonroot
COPY --from=builder /app/target/release/myapp /
ENTRYPOINT ["/myapp"]
```

### No Secrets in Layers

Every `COPY`, `ADD`, and `RUN` instruction creates a layer. Secrets added in one layer persist even if deleted in a subsequent layer.

```dockerfile
# ❌ Bad: Secret persists in image layer history
COPY .env /app/.env
RUN source /app/.env && build.sh
RUN rm /app/.env          # Secret is still in the previous layer!

# ❌ Bad: Secret in build arg (visible in image history)
ARG DATABASE_URL
RUN echo $DATABASE_URL > /app/config

# ✅ Good: Use BuildKit secrets (not stored in any layer)
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=db_url \
    export DATABASE_URL=$(cat /run/secrets/db_url) && \
    ./configure --db-url="$DATABASE_URL"
```

```bash
# Build with secret
docker build --secret id=db_url,src=./db_url.txt -t myapp .
```

### Pin Versions

Unpinned versions introduce non-reproducible builds and potential supply chain attacks.

```dockerfile
# ❌ Bad: Unpinned base image and packages
FROM python:latest
RUN pip install flask requests

# ✅ Good: Pinned base image and packages
FROM python:3.12.3-alpine3.19@sha256:abc123...
RUN pip install --no-cache-dir flask==3.0.2 requests==2.31.0

# ✅ Good: Pinned system packages
FROM debian:bookworm-20240311-slim
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      curl=7.88.1-10+deb12u5 \
      ca-certificates=20230311 && \
    rm -rf /var/lib/apt/lists/*
```

---

## Runtime Security

### Read-Only Filesystems

Mount the container's root filesystem as read-only to prevent runtime modification.

```bash
# Run with read-only root filesystem
docker run --read-only myapp:latest

# Allow writes only to specific directories
docker run --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --tmpfs /var/run:rw,noexec,nosuid,size=1m \
  myapp:latest
```

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:latest
    read_only: true
    tmpfs:
      - /tmp:size=64m
      - /var/run:size=1m
```

### Dropping Capabilities

Docker grants containers a default set of Linux capabilities. Drop all and add only what is needed.

```bash
# See default capabilities
docker run --rm alpine cat /proc/1/status | grep Cap

# Drop ALL capabilities, add only what is needed
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp:latest

# Common capability sets for web applications
docker run \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --cap-add=CHOWN \
  --cap-add=SETUID \
  --cap-add=SETGID \
  myapp:latest
```

| Capability | Description | Typically Needed |
|---|---|---|
| `NET_BIND_SERVICE` | Bind to ports below 1024 | Yes, for port 80/443 |
| `CHOWN` | Change file ownership | Sometimes, at startup |
| `SETUID` / `SETGID` | Change process user/group | Sometimes, for init |
| `SYS_ADMIN` | Broad admin privileges | ❌ Never in production |
| `NET_RAW` | Use raw sockets (ping) | Rarely |
| `MKNOD` | Create device files | ❌ Not for apps |
| `SYS_PTRACE` | Trace processes | ❌ Debugging only |

### Seccomp Profiles

Seccomp (secure computing mode) restricts which system calls a container can make.

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": [
        "accept4", "bind", "clone", "close", "connect",
        "epoll_ctl", "epoll_wait", "execve", "exit_group",
        "fcntl", "fstat", "futex", "getpid", "listen",
        "mmap", "mprotect", "munmap", "openat", "read",
        "recvfrom", "sendto", "setsockopt", "socket",
        "write"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

```bash
# Apply a custom seccomp profile
docker run --security-opt seccomp=custom-profile.json myapp:latest

# Use Docker's default seccomp profile (blocks ~44 syscalls)
docker run myapp:latest   # default profile applied automatically

# Disable seccomp (NOT recommended)
docker run --security-opt seccomp=unconfined myapp:latest
```

### AppArmor and SELinux

**AppArmor** (Ubuntu/Debian):

```bash
# Check the default Docker AppArmor profile
docker run --rm alpine cat /proc/1/attr/current
# Output: docker-default (enforce)

# Apply a custom AppArmor profile
sudo apparmor_parser -r /etc/apparmor.d/docker-custom
docker run --security-opt apparmor=docker-custom myapp:latest
```

**SELinux** (RHEL/CentOS/Fedora):

```bash
# Run with SELinux labels
docker run --security-opt label=type:container_t myapp:latest

# Disable SELinux for a container (not recommended)
docker run --security-opt label=disable myapp:latest

# Check SELinux context
docker inspect --format '{{.HostConfig.SecurityOpt}}' <container-id>
```

---

## Rootless Containers

Rootless containers run the Docker daemon and containers entirely without root privileges, eliminating the largest class of container escape vulnerabilities.

### Docker Rootless Mode

```bash
# Install rootless Docker (requires newuidmap/newgidmap)
dockerd-rootless-setuptool.sh install

# Set environment variables (add to ~/.bashrc)
export PATH=$HOME/bin:$PATH
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock

# Verify rootless mode
docker info | grep "rootless"
# Security Options: rootless

# Run a container (runs as your user, not root)
docker run -d -p 8080:80 nginx:alpine
```

> **Note:** Rootless mode has limitations — it cannot bind to ports below 1024 without extra configuration, and some storage drivers are not available.

### Podman

Podman is a daemonless, rootless container engine that is CLI-compatible with Docker.

```bash
# Run a container rootless (no daemon required)
podman run -d -p 8080:80 nginx:alpine

# Build an image
podman build -t myapp .

# Generate a systemd unit file for a rootless container
podman generate systemd --name mycontainer --files --new

# Podman with user namespace mapping
podman run --userns=auto myapp:latest
```

| Feature | Docker Rootless | Podman |
|---|---|---|
| Daemon required | Yes (rootless daemon) | No (daemonless) |
| Default mode | Opt-in | Rootless by default |
| CLI compatibility | Docker CLI | Docker-compatible |
| Compose support | docker-compose | podman-compose / Quadlet |
| Kubernetes YAML | ❌ | ✅ `podman play kube` |
| Systemd integration | Limited | ✅ Quadlet / generate |

---

## Secrets Management

### Docker Secrets

Docker Swarm provides built-in secrets management. Secrets are encrypted at rest and only available to services that explicitly request them.

```bash
# Create a secret
echo "supersecretpassword" | docker secret create db_password -

# Create a secret from a file
docker secret create tls_cert ./server.crt

# Use secrets in a service
docker service create \
  --name web \
  --secret db_password \
  --secret tls_cert \
  myapp:latest
```

Inside the container, secrets are mounted at `/run/secrets/<secret_name>`:

```python
# Reading a Docker secret in application code
import pathlib

db_password = pathlib.Path("/run/secrets/db_password").read_text().strip()
```

### Build-Time vs Runtime Secrets

| Aspect | Build-Time Secret | Runtime Secret |
|---|---|---|
| **Purpose** | Authenticate during image build (e.g., private registry, npm token) | Configure running containers (e.g., DB password, API key) |
| **Mechanism** | `docker build --secret` (BuildKit) | Environment variables, mounted files, vault agents |
| **Persists in image** | ❌ No (if using BuildKit secrets) | N/A |
| **Example** | Private npm registry token for `npm install` | Database connection string |

```dockerfile
# Build-time secret example (BuildKit)
# syntax=docker/dockerfile:1
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci --only=production
COPY . .
USER node
CMD ["node", "server.js"]
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t myapp .
```

### External Vault Integration

```yaml
# docker-compose.yml with HashiCorp Vault agent sidecar
services:
  vault-agent:
    image: hashicorp/vault:1.15
    command: ["vault", "agent", "-config=/vault/config.hcl"]
    volumes:
      - vault-secrets:/vault/secrets
      - ./vault-agent-config.hcl:/vault/config.hcl:ro

  app:
    image: myapp:latest
    depends_on:
      - vault-agent
    volumes:
      - vault-secrets:/secrets:ro
    environment:
      - SECRETS_PATH=/secrets

volumes:
  vault-secrets:
```

---

## Network Security

### Network Segmentation

Isolate containers by placing them on separate Docker networks. Containers on different networks cannot communicate by default.

```bash
# Create isolated networks for different tiers
docker network create --driver bridge frontend
docker network create --driver bridge backend
docker network create --driver bridge database --internal

# --internal prevents external access (no internet)

# Run containers on appropriate networks
docker run -d --name web --network frontend nginx
docker run -d --name api --network backend myapi
docker run -d --name db --network database postgres:16

# Connect the API to both frontend and backend
docker network connect frontend api
```

```
Network Segmentation
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Internet
     │
     ▼
  ┌──────────────────────┐
  │   Frontend Network   │
  │  ┌─────┐   ┌─────┐  │
  │  │ Web │   │ API │  │
  │  └─────┘   └──┬──┘  │
  └───────────────┼──────┘
                  │ (API on both networks)
  ┌───────────────┼──────┐
  │   Backend Network    │
  │              ┌┴────┐ │
  │              │ API │ │
  │              └──┬──┘ │
  └─────────────────┼────┘
                    │
  ┌─────────────────┼────┐
  │  Database Network    │  ← --internal (no internet)
  │              ┌──┴──┐ │
  │              │ DB  │ │
  │              └─────┘ │
  └──────────────────────┘
```

### TLS Between Containers

Encrypt traffic between containers, especially across untrusted networks.

```yaml
# docker-compose.yml with TLS sidecar (Nginx as TLS terminator)
services:
  app:
    image: myapp:latest
    networks:
      - internal

  tls-proxy:
    image: nginx:alpine
    volumes:
      - ./tls/server.crt:/etc/nginx/certs/server.crt:ro
      - ./tls/server.key:/etc/nginx/certs/server.key:ro
      - ./nginx-tls.conf:/etc/nginx/conf.d/default.conf:ro
    ports:
      - "443:443"
    networks:
      - internal
      - external

networks:
  internal:
    internal: true
  external:
```

### Docker Socket Protection

The Docker socket (`/var/run/docker.sock`) provides full root access to the host. Mounting it into a container is equivalent to giving that container root on the host.

```bash
# ❌ DANGEROUS: Never do this in production
docker run -v /var/run/docker.sock:/var/run/docker.sock myapp

# ✅ If Docker socket access is absolutely required, use a proxy
# docker-socket-proxy limits which API endpoints are accessible
docker run -d \
  --name socket-proxy \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -e CONTAINERS=1 \
  -e IMAGES=0 \
  -e EXEC=0 \
  -e VOLUMES=0 \
  -p 2375:2375 \
  tecnativa/docker-socket-proxy
```

---

## Supply Chain Security

### SBOM Generation

A Software Bill of Materials (SBOM) lists all components in your container image, enabling vulnerability tracking and license compliance.

```bash
# Generate SBOM with Syft
syft myapp:latest -o spdx-json > sbom.spdx.json
syft myapp:latest -o cyclonedx-json > sbom.cdx.json

# Attach SBOM to an image with cosign
cosign attach sbom --sbom sbom.cdx.json myregistry.io/myapp:v1.0.0

# Generate SBOM with Docker (BuildKit)
docker buildx build --sbom=true -t myapp:latest .

# Scan an SBOM for vulnerabilities
grype sbom:sbom.cdx.json
```

### Image Provenance

Provenance records document how an image was built — the source repository, build command, builder identity, and build parameters.

```bash
# Build with provenance attestation (BuildKit)
docker buildx build \
  --provenance=true \
  --push \
  -t myregistry.io/myapp:v1.0.0 .

# Inspect provenance
cosign verify-attestation \
  --type slsaprovenance \
  --certificate-identity=ci@myorg.com \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  myregistry.io/myapp:v1.0.0
```

### Sigstore and Cosign Workflows

```
Supply Chain Security Workflow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Source Code
      │
      ▼
  ┌──────────┐     ┌──────────┐     ┌───────────┐
  │  Build   │────▶│  Sign    │────▶│  Push to  │
  │  Image   │     │  (cosign)│     │  Registry │
  └──────────┘     └──────────┘     └───────────┘
      │                 │                 │
      ▼                 ▼                 ▼
  ┌──────────┐     ┌──────────┐     ┌───────────┐
  │  Gen     │     │  Record  │     │  Verify   │
  │  SBOM    │     │  in      │     │  before   │
  │  (syft)  │     │  Rekor   │     │  deploy   │
  └──────────┘     └──────────┘     └───────────┘
```

---

## Docker Daemon Security

### TLS for Remote API

If the Docker daemon is exposed over the network, TLS with mutual authentication is mandatory.

```bash
# Generate CA, server, and client certificates
openssl genrsa -aes256 -out ca-key.pem 4096
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem

# Configure dockerd with TLS
dockerd \
  --tlsverify \
  --tlscacert=ca.pem \
  --tlscert=server-cert.pem \
  --tlskey=server-key.pem \
  -H=0.0.0.0:2376

# Connect with TLS client
docker --tlsverify \
  --tlscacert=ca.pem \
  --tlscert=client-cert.pem \
  --tlskey=client-key.pem \
  -H=tcp://dockerhost:2376 info
```

### User Namespaces

User namespace remapping maps container root (UID 0) to an unprivileged user on the host.

```json
// /etc/docker/daemon.json
{
  "userns-remap": "default"
}
```

```bash
# Verify user namespace mapping
cat /etc/subuid
# dockremap:100000:65536

cat /etc/subgid
# dockremap:100000:65536

# Restart Docker daemon
sudo systemctl restart docker

# Verify — container root maps to unprivileged host user
docker run --rm alpine id
# uid=0(root) gid=0(root)   ← inside container

# On the host, the process runs as UID 100000
ps aux | grep "alpine"
```

### Authorization Plugins

Authorization plugins provide fine-grained access control for Docker API requests.

```json
// /etc/docker/daemon.json — enable OPA authorization plugin
{
  "authorization-plugins": ["openpolicyagent/opa-docker-authz-v2:0.9"]
}
```

```rego
# OPA policy: deny running privileged containers
package docker.authz

default allow = false

allow {
    not is_privileged
}

is_privileged {
    input.Body.HostConfig.Privileged == true
}
```

---

## Container Scanning in CI/CD

### GitHub Actions Example

```yaml
# .github/workflows/container-security.yml
name: Container Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif

      - name: Run Dockle linter
        run: |
          VERSION=$(curl --silent "https://api.github.com/repos/goodwithtech/dockle/releases/latest" | grep '"tag_name":' | sed -E 's/.*"v([^"]+)".*/\1/')
          curl -sL "https://github.com/goodwithtech/dockle/releases/download/v${VERSION}/dockle_${VERSION}_Linux-64bit.tar.gz" | tar xz
          ./dockle --exit-code 1 --exit-level warn myapp:${{ github.sha }}
```

### GitLab CI Example

```yaml
# .gitlab-ci.yml
container_scanning:
  stage: test
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  allow_failure: false
```

---

## Security Checklist

| Category | Check | Priority |
|---|---|---|
| **Image** | Use minimal base images (alpine, distroless, scratch) | 🔴 Critical |
| **Image** | Scan images for vulnerabilities in CI/CD | 🔴 Critical |
| **Image** | Pin base image versions and package versions | 🟠 High |
| **Image** | Sign images with cosign or DCT | 🟠 High |
| **Dockerfile** | Run as non-root USER | 🔴 Critical |
| **Dockerfile** | No secrets in image layers (use BuildKit `--secret`) | 🔴 Critical |
| **Dockerfile** | Use multi-stage builds to minimize final image | 🟠 High |
| **Dockerfile** | Use `COPY` instead of `ADD` | 🟡 Medium |
| **Runtime** | Drop all capabilities, add only required ones | 🔴 Critical |
| **Runtime** | Use read-only root filesystem | 🟠 High |
| **Runtime** | Never run with `--privileged` | 🔴 Critical |
| **Runtime** | Apply seccomp profiles | 🟠 High |
| **Runtime** | Set memory and CPU limits | 🟠 High |
| **Network** | Segment containers into isolated networks | 🟠 High |
| **Network** | Never expose the Docker socket | 🔴 Critical |
| **Network** | Enable TLS for Docker daemon remote access | 🔴 Critical |
| **Supply Chain** | Generate and attach SBOMs to images | 🟠 High |
| **Supply Chain** | Verify image signatures before deployment | 🟠 High |
| **Supply Chain** | Use private/trusted registries | 🟠 High |
| **Secrets** | Use Docker secrets or external vaults | 🔴 Critical |
| **Secrets** | Never store secrets in environment variables in Dockerfiles | 🔴 Critical |
| **Host** | Enable user namespace remapping | 🟠 High |
| **Host** | Keep the host kernel and Docker daemon patched | 🔴 Critical |
| **Host** | Consider rootless Docker or Podman | 🟠 High |

---

## Next Steps

Continue your Docker learning journey:

| File | Topic | Description |
|---|---|---|
| [08-PERFORMANCE.md](08-PERFORMANCE.md) | Docker Performance | Resource limits, image optimization, build performance, and monitoring |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Docker Best Practices | Production patterns, health checks, logging, CI/CD integration |
| [06-REGISTRIES.md](06-REGISTRIES.md) | Container Registries | Registry security, image signing, and authentication patterns |
| [01-IMAGES-AND-DOCKERFILES.md](01-IMAGES-AND-DOCKERFILES.md) | Images & Dockerfiles | Multi-stage builds, Dockerfile instructions, and image layering |
| [02-CONTAINER-RUNTIME.md](02-CONTAINER-RUNTIME.md) | Container Runtime | Docker Engine, containerd, runc, and container lifecycle |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Docker Security documentation |
