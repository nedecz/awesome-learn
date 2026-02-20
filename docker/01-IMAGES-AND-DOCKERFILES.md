# Docker Images and Dockerfiles

## Table of Contents

1. [Overview](#overview)
   - [What Is a Docker Image](#what-is-a-docker-image)
   - [Image Layers](#image-layers)
   - [The Build Process](#the-build-process)
2. [Dockerfile Instructions Reference](#dockerfile-instructions-reference)
   - [FROM](#from)
   - [RUN](#run)
   - [COPY](#copy)
   - [ADD](#add)
   - [WORKDIR](#workdir)
   - [EXPOSE](#expose)
   - [ENV](#env)
   - [ARG](#arg)
   - [CMD](#cmd)
   - [ENTRYPOINT](#entrypoint)
   - [HEALTHCHECK](#healthcheck)
   - [USER](#user)
   - [LABEL](#label)
   - [VOLUME](#volume)
   - [SHELL](#shell)
3. [CMD vs ENTRYPOINT](#cmd-vs-entrypoint)
4. [Multi-Stage Builds](#multi-stage-builds)
   - [Why Multi-Stage Builds](#why-multi-stage-builds)
   - [Go Application Example](#go-application-example)
   - [Node.js Application Example](#nodejs-application-example)
5. [Build Context and .dockerignore](#build-context-and-dockerignore)
6. [Image Layer Optimization](#image-layer-optimization)
   - [Ordering Instructions](#ordering-instructions)
   - [Combining RUN Commands](#combining-run-commands)
   - [Leveraging Build Cache](#leveraging-build-cache)
7. [Tagging Strategies](#tagging-strategies)
   - [Semantic Versioning](#semantic-versioning)
   - [Git SHA Tags](#git-sha-tags)
   - [The latest Tag Pitfalls](#the-latest-tag-pitfalls)
8. [BuildKit Features](#buildkit-features)
   - [Cache Mounts](#cache-mounts)
   - [Secret Mounts](#secret-mounts)
   - [SSH Mounts](#ssh-mounts)
9. [Base Image Selection](#base-image-selection)
10. [Best Practices Summary](#best-practices-summary)
11. [Next Steps](#next-steps)
12. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **Docker images** and **Dockerfiles** — from the fundamental concepts of image layers and the build process, through every Dockerfile instruction, to advanced topics like multi-stage builds, BuildKit features, and image optimization strategies.

### Target Audience

- **Developers** writing Dockerfiles to package their applications
- **DevOps Engineers** optimizing image builds for CI/CD pipelines
- **SREs** troubleshooting image size, security, and runtime behavior
- **Architects** defining base image standards and build strategies

### Scope

- Docker image internals: layers, manifests, and the content-addressable store
- Complete Dockerfile instruction reference with practical examples
- Multi-stage builds for compiled and interpreted languages
- Build context management and `.dockerignore` patterns
- Layer optimization techniques for smaller, faster builds
- Tagging strategies for production image management
- BuildKit advanced features: cache mounts, secrets, and SSH forwarding
- Base image selection criteria and comparison

### What Is a Docker Image

A Docker image is an **ordered collection of read-only filesystem layers** plus metadata that together contain everything needed to run an application — code, runtime, libraries, environment variables, and configuration files.

Images are built from a **Dockerfile**, a text file containing a sequence of instructions. Each instruction creates a new layer in the image. Layers are stacked on top of each other using a **union filesystem** (typically OverlayFS), presenting a unified view to the container.

```
┌─────────────────────────────────────────────────────────┐
│                  Container (read-write layer)             │
│              Writable — changes stored here               │
├─────────────────────────────────────────────────────────┤
│  Layer 4:  CMD ["node", "server.js"]                     │  ← Metadata only
├─────────────────────────────────────────────────────────┤
│  Layer 3:  COPY . /app                                   │  ← Application code
├─────────────────────────────────────────────────────────┤
│  Layer 2:  RUN npm ci --production                       │  ← Dependencies
├─────────────────────────────────────────────────────────┤
│  Layer 1:  FROM node:20-alpine                           │  ← Base OS + Node.js
└─────────────────────────────────────────────────────────┘
```

### Image Layers

Every Dockerfile instruction that modifies the filesystem creates a new **layer**. Layers are:

- **Immutable** — once created, a layer never changes
- **Content-addressable** — identified by a SHA256 digest of their contents
- **Shared** — if two images use the same base image, they share those layers on disk and in registry transfers
- **Cached** — Docker reuses layers from previous builds when the instruction and its context have not changed

```
Image: myapp:1.0
  ├── sha256:a1b2c3...  (base OS layer — 5 MB)
  ├── sha256:d4e5f6...  (apt install layer — 45 MB)
  ├── sha256:789abc...  (npm install layer — 120 MB)
  └── sha256:def012...  (COPY app source — 2 MB)
                          ─────────────────────
                          Total: ~172 MB

Image: myapp:1.1  (only app code changed)
  ├── sha256:a1b2c3...  (shared — not re-downloaded)
  ├── sha256:d4e5f6...  (shared — not re-downloaded)
  ├── sha256:789abc...  (shared — not re-downloaded)
  └── sha256:aaa111...  (new layer — 2 MB)
                          ─────────────────────
                          Only 2 MB transferred on push/pull
```

### The Build Process

When you run `docker build`, the following sequence occurs:

1. **Send build context** — Docker client sends the build context (files in the directory) to the daemon
2. **Parse Dockerfile** — The daemon reads and validates the Dockerfile
3. **Execute instructions** — Each instruction runs in order, creating intermediate layers
4. **Cache check** — For each instruction, Docker checks if a cached layer already exists
5. **Produce final image** — The final image is tagged and stored in the local image store

```bash
# Basic build command
docker build -t myapp:1.0 .

# Build with a specific Dockerfile
docker build -f Dockerfile.production -t myapp:1.0 .

# Build with build arguments
docker build --build-arg NODE_ENV=production -t myapp:1.0 .

# Build with no cache
docker build --no-cache -t myapp:1.0 .
```

---

## Dockerfile Instructions Reference

### FROM

Sets the **base image** for subsequent instructions. Every Dockerfile must begin with a `FROM` instruction (or `ARG` before `FROM`).

```dockerfile
# Use an official base image
FROM node:20-alpine

# Use a specific digest for reproducibility
FROM node@sha256:a1b2c3d4e5f6...

# Use a scratch (empty) base for static binaries
FROM scratch

# Name a build stage for multi-stage builds
FROM golang:1.22 AS builder
```

### RUN

Executes a command in a new layer on top of the current image and commits the result.

```dockerfile
# Shell form (runs in /bin/sh -c)
RUN apt-get update && apt-get install -y curl

# Exec form (no shell processing)
RUN ["apt-get", "install", "-y", "curl"]

# Multi-line for readability
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        ca-certificates \
        git && \
    rm -rf /var/lib/apt/lists/*
```

### COPY

Copies files or directories from the build context into the image filesystem.

```dockerfile
# Copy a single file
COPY package.json /app/package.json

# Copy everything from the build context
COPY . /app/

# Copy with a specific owner
COPY --chown=node:node . /app/

# Copy from a named build stage
COPY --from=builder /app/binary /usr/local/bin/binary
```

### ADD

Similar to `COPY` but with additional features: automatic tar extraction and URL support. **Prefer `COPY` unless you specifically need these features.**

```dockerfile
# ADD auto-extracts local tar archives
ADD app.tar.gz /app/

# ADD can fetch remote URLs (but prefer curl in RUN for caching)
ADD https://example.com/file.txt /app/file.txt
```

### WORKDIR

Sets the working directory for subsequent `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, and `ADD` instructions.

```dockerfile
WORKDIR /app

# WORKDIR creates the directory if it does not exist
WORKDIR /app/src

# Multiple WORKDIRs are resolved relative to the previous one
WORKDIR /app
WORKDIR src
# Working directory is now /app/src
```

### EXPOSE

Documents which ports the container listens on at runtime. This is **metadata only** — it does not actually publish the port.

```dockerfile
# Expose a single port
EXPOSE 8080

# Expose multiple ports
EXPOSE 8080 8443

# Expose UDP port
EXPOSE 53/udp
```

### ENV

Sets environment variables that persist in the running container.

```dockerfile
# Single variable
ENV NODE_ENV=production

# Multiple variables
ENV APP_HOME=/app \
    APP_PORT=8080

# ENV values are available to subsequent instructions
ENV PATH="/app/bin:${PATH}"
```

### ARG

Defines build-time variables that users can pass with `--build-arg`. ARG values are **not available** in the running container (use `ENV` for that).

```dockerfile
# Define a build argument with a default value
ARG NODE_VERSION=20

# Use the argument
FROM node:${NODE_VERSION}-alpine

# ARG after FROM must be re-declared
ARG BUILD_DATE
LABEL build-date=${BUILD_DATE}
```

### CMD

Sets the **default command** to run when the container starts. Can be overridden by passing arguments to `docker run`.

```dockerfile
# Exec form (preferred — no shell processing, signals go directly to process)
CMD ["node", "server.js"]

# Shell form (runs as /bin/sh -c)
CMD node server.js

# As default parameters to ENTRYPOINT
CMD ["--port", "8080"]
```

### ENTRYPOINT

Configures the container to run as an executable. Unlike `CMD`, it is **not easily overridden** by `docker run` arguments (those become arguments to the ENTRYPOINT).

```dockerfile
# Exec form (preferred)
ENTRYPOINT ["python", "app.py"]

# Combined with CMD for default arguments
ENTRYPOINT ["python", "app.py"]
CMD ["--host", "0.0.0.0", "--port", "8080"]

# Shell form (not recommended — PID 1 will be /bin/sh, not your process)
ENTRYPOINT python app.py
```

### HEALTHCHECK

Tells Docker how to test that the container is still working. Docker uses the health check to determine the container's health status.

```dockerfile
# HTTP health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Custom health check script
HEALTHCHECK --interval=15s --timeout=3s --retries=5 \
    CMD /app/healthcheck.sh

# Disable health check inherited from base image
HEALTHCHECK NONE
```

| Parameter | Default | Description |
|---|---|---|
| `--interval` | 30s | Time between health checks |
| `--timeout` | 30s | Maximum time a check can take before considered failed |
| `--start-period` | 0s | Grace period for container startup before checks count |
| `--retries` | 3 | Number of consecutive failures before marking unhealthy |

### USER

Sets the user (and optionally group) for subsequent `RUN`, `CMD`, and `ENTRYPOINT` instructions. **Always run production containers as non-root.**

```dockerfile
# Create a non-root user and switch to it
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Use UID:GID for portability
USER 1001:1001
```

### LABEL

Adds metadata to the image as key-value pairs. Labels are used for organization, automation, and tooling.

```dockerfile
LABEL maintainer="platform-engineering@company.com"
LABEL version="1.0.0"
LABEL description="Production API server"

# OCI standard labels
LABEL org.opencontainers.image.title="myapp"
LABEL org.opencontainers.image.version="1.0.0"
LABEL org.opencontainers.image.source="https://github.com/org/myapp"
LABEL org.opencontainers.image.created="2025-01-15T10:30:00Z"
```

### VOLUME

Creates a mount point and marks it as holding externally mounted volumes from the host or other containers.

```dockerfile
# Declare a volume mount point
VOLUME /data

# Multiple volumes
VOLUME ["/data", "/logs"]
```

> **Note:** `VOLUME` in a Dockerfile creates an anonymous volume at runtime. For production, prefer explicit named volumes or bind mounts in `docker run` or Compose files.

### SHELL

Overrides the default shell used for the shell form of `RUN`, `CMD`, and `ENTRYPOINT`.

```dockerfile
# Default on Linux is ["/bin/sh", "-c"]
# Default on Windows is ["cmd", "/S", "/C"]

# Switch to bash
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Now RUN commands use bash
RUN echo "hello" | grep "hello"
```

---

## CMD vs ENTRYPOINT

Understanding the interaction between `CMD` and `ENTRYPOINT` is critical for writing correct Dockerfiles.

| Scenario | ENTRYPOINT | CMD | `docker run myapp` | `docker run myapp --debug` |
|---|---|---|---|---|
| CMD only | _(none)_ | `["node", "app.js"]` | `node app.js` | `--debug` (replaces CMD) |
| ENTRYPOINT only | `["node", "app.js"]` | _(none)_ | `node app.js` | `node app.js --debug` |
| Both (recommended) | `["node", "app.js"]` | `["--port", "8080"]` | `node app.js --port 8080` | `node app.js --debug` |

**Guidelines:**

- Use **`CMD`** when you want the default command to be easily overridable
- Use **`ENTRYPOINT`** when the container should always run a specific executable
- Use **both** when you want a fixed executable with default arguments that users can override

```dockerfile
# Pattern: ENTRYPOINT + CMD for a CLI tool
FROM python:3.12-slim
COPY cli.py /app/cli.py
WORKDIR /app
ENTRYPOINT ["python", "cli.py"]
CMD ["--help"]

# docker run mycli            → python cli.py --help
# docker run mycli --version  → python cli.py --version
# docker run mycli process    → python cli.py process
```

```dockerfile
# Pattern: CMD only for a web server
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm ci --production
CMD ["node", "server.js"]

# docker run myapp                        → node server.js
# docker run myapp node scripts/migrate   → node scripts/migrate
```

---

## Multi-Stage Builds

### Why Multi-Stage Builds

Multi-stage builds allow you to use multiple `FROM` statements in a single Dockerfile. Each `FROM` begins a new build stage. You can selectively copy artifacts from one stage to another, leaving behind everything you do not need in the final image.

```
┌─────────────────────────┐     ┌─────────────────────────┐
│     Stage 1: builder     │     │   Stage 2: production    │
│                         │     │                         │
│  Go toolchain (800 MB)  │     │  scratch / alpine (5 MB)│
│  Source code            │────►│  Compiled binary (10 MB) │
│  Dependencies           │     │                         │
│  Test tools             │     │  Total: ~15 MB          │
│                         │     │                         │
│  Total: ~1.2 GB        │     └─────────────────────────┘
└─────────────────────────┘
        discarded                      shipped
```

**Benefits:**

- **Dramatically smaller images** — final image contains only the runtime and binary
- **Better security** — no compilers, package managers, or build tools in production
- **Single Dockerfile** — no need for separate build and runtime Dockerfiles
- **Reproducible builds** — build environment is defined in the same file

### Go Application Example

```dockerfile
# ── Stage 1: Build ──────────────────────────────────────
FROM golang:1.22-alpine AS builder

# Install build dependencies
RUN apk add --no-cache git ca-certificates

WORKDIR /src

# Copy go.mod and go.sum first for better layer caching
COPY go.mod go.sum ./
RUN go mod download

# Copy source code and build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -o /app/server ./cmd/server

# ── Stage 2: Runtime ────────────────────────────────────
FROM scratch

# Import CA certificates from builder
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy the compiled binary
COPY --from=builder /app/server /server

# Run as non-root (numeric UID since scratch has no /etc/passwd)
USER 65534:65534

EXPOSE 8080
ENTRYPOINT ["/server"]
```

```bash
# Build the image
docker build -t myapp:1.0 .

# Check the final image size
docker images myapp:1.0
# REPOSITORY   TAG   IMAGE ID       CREATED          SIZE
# myapp        1.0   abc123def456   10 seconds ago   12.4MB
```

### Node.js Application Example

```dockerfile
# ── Stage 1: Install dependencies ───────────────────────
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production

# ── Stage 2: Build (for TypeScript / bundled apps) ──────
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# ── Stage 3: Production runtime ─────────────────────────
FROM node:20-alpine AS production

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy production dependencies from deps stage
COPY --from=deps --chown=appuser:appgroup /app/node_modules ./node_modules

# Copy built application from builder stage
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

USER appuser

EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

---

## Build Context and .dockerignore

The **build context** is the set of files and directories sent to the Docker daemon when you run `docker build`. By default, it is the directory you specify (usually `.`).

A large build context slows down builds because every file is sent to the daemon before any instruction executes. Use a `.dockerignore` file to exclude files that are not needed in the image.

```
Build Context Flow:
                                    ┌──────────────────┐
  Project Directory                 │   Docker Daemon    │
  ┌─────────────────┐    tar.gz    │                    │
  │  src/           │──────────────►│  1. Receive context│
  │  package.json   │              │  2. Parse Dockerfile│
  │  node_modules/  │ ✘ excluded   │  3. Execute layers  │
  │  .git/          │ ✘ excluded   │  4. Produce image   │
  │  Dockerfile     │              │                    │
  │  .dockerignore  │              └──────────────────────┘
  └─────────────────┘
```

### .dockerignore File

```
# .dockerignore

# Version control
.git
.gitignore

# Dependencies (installed inside the container)
node_modules
vendor

# Build output
dist
build
*.o
*.exe

# IDE and editor files
.vscode
.idea
*.swp
*.swo

# Docker files not needed in context
Dockerfile*
docker-compose*.yml
.dockerignore

# CI/CD configuration
.github
.gitlab-ci.yml
Jenkinsfile

# Documentation
*.md
LICENSE
docs/

# Environment files with secrets
.env
.env.*
*.pem
*.key
```

---

## Image Layer Optimization

### Ordering Instructions

Docker invalidates the cache for a layer and **all subsequent layers** when an instruction or its inputs change. Place instructions that change least frequently at the top and most frequently at the bottom.

```dockerfile
# ✅ GOOD — dependencies change less often than source code
FROM node:20-alpine
WORKDIR /app

# Layer 1: rarely changes
COPY package.json package-lock.json ./

# Layer 2: only rebuilds when package files change
RUN npm ci --production

# Layer 3: changes with every code commit
COPY . .

CMD ["node", "server.js"]
```

```dockerfile
# ❌ BAD — copying everything first invalidates npm install cache on every code change
FROM node:20-alpine
WORKDIR /app

COPY . .
RUN npm ci --production

CMD ["node", "server.js"]
```

### Combining RUN Commands

Each `RUN` instruction creates a new layer. Combine related commands to reduce layers and avoid leaving unnecessary files in intermediate layers.

```dockerfile
# ❌ BAD — three layers; apt cache remains in first layer even after rm
RUN apt-get update
RUN apt-get install -y curl git
RUN rm -rf /var/lib/apt/lists/*

# ✅ GOOD — single layer; apt cache is removed in the same layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        git && \
    rm -rf /var/lib/apt/lists/*
```

### Leveraging Build Cache

```
Instruction unchanged + context unchanged → cache HIT  (instant)
Instruction unchanged + context changed  → cache MISS (rebuild this + all below)
Instruction changed                      → cache MISS (rebuild this + all below)
```

**Tips for maximizing cache hits:**

1. **Pin base image versions** — `FROM node:20.11-alpine` not `FROM node:latest`
2. **Copy dependency files first** — `COPY package.json` before `COPY .`
3. **Use `--mount=type=cache`** with BuildKit for package manager caches
4. **Avoid `COPY . .` early** — it invalidates cache on any file change
5. **Sort multi-line arguments** — alphabetical sorting reduces diff noise

```dockerfile
# Alphabetical sorting prevents unnecessary cache invalidation
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        git \
        jq \
        wget && \
    rm -rf /var/lib/apt/lists/*
```

---

## Tagging Strategies

Image tags are mutable references to specific image digests. A well-defined tagging strategy is critical for reproducible deployments.

### Semantic Versioning

```bash
# Tag with full semantic version
docker build -t myapp:1.2.3 .

# Also tag major and minor for convenience
docker tag myapp:1.2.3 myapp:1.2
docker tag myapp:1.2.3 myapp:1

# Push all tags
docker push myapp:1.2.3
docker push myapp:1.2
docker push myapp:1
```

| Tag | Stability | Use Case |
|---|---|---|
| `myapp:1.2.3` | Immutable (by convention) | Production deployments — exact version |
| `myapp:1.2` | Floats to latest patch | Staging — get latest bug fixes |
| `myapp:1` | Floats to latest minor | Development — get latest features in major |

### Git SHA Tags

```bash
# Tag with the short Git SHA for traceability
GIT_SHA=$(git rev-parse --short HEAD)
docker build -t myapp:${GIT_SHA} .

# Combine with semver in CI/CD
docker build \
    -t myapp:1.2.3 \
    -t myapp:${GIT_SHA} \
    -t myapp:latest .
```

### The latest Tag Pitfalls

The `latest` tag is the **default tag** applied when no tag is specified. It is **not** a special tag — Docker does not automatically update it.

| Pitfall | Description |
|---|---|
| **Not actually latest** | `latest` only points to whatever was last pushed without a tag; it may be stale |
| **Not reproducible** | `docker pull myapp:latest` can return different images on different days |
| **Breaks rollbacks** | If you deploy `latest` and need to rollback, there is no previous version to reference |
| **Cache confusion** | `FROM myapp:latest` in a Dockerfile may use a cached old version |

**Recommendation:** Never use `latest` in production. Always use explicit version tags or digests.

```bash
# ✅ Production — use exact digest for maximum reproducibility
docker pull myapp@sha256:a1b2c3d4e5f6...

# ✅ Production — use semver tag
docker pull myapp:1.2.3

# ❌ Production — avoid latest
docker pull myapp:latest
```

---

## BuildKit Features

BuildKit is the modern builder backend for Docker, enabled by default in Docker Desktop and Docker Engine 23.0+. It provides significant improvements over the legacy builder.

```bash
# Enable BuildKit (if not default)
export DOCKER_BUILDKIT=1

# Or use docker buildx
docker buildx build -t myapp:1.0 .
```

### Cache Mounts

Cache mounts allow you to persist package manager caches across builds, dramatically speeding up dependency installation.

```dockerfile
# syntax=docker/dockerfile:1

# Cache apt packages
FROM ubuntu:22.04
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends curl git

# Cache Go modules
FROM golang:1.22
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /app/server .

# Cache npm packages
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --production
```

### Secret Mounts

Secret mounts allow you to pass sensitive data (API keys, tokens) to the build without embedding them in image layers.

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.12-slim

# The secret is mounted at /run/secrets/pypi_token during this RUN only
# It is never written to any image layer
RUN --mount=type=secret,id=pypi_token \
    pip install --index-url https://$(cat /run/secrets/pypi_token)@pypi.example.com/simple/ \
    private-package==1.0.0
```

```bash
# Pass the secret at build time
docker build --secret id=pypi_token,src=./pypi_token.txt -t myapp:1.0 .
```

### SSH Mounts

SSH mounts forward the host's SSH agent to the build container, enabling private repository access without copying keys.

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.22 AS builder
WORKDIR /src

# Configure Git to use SSH for private modules
RUN git config --global url."git@github.com:".insteadOf "https://github.com/"

# Mount SSH agent and download private dependencies
RUN --mount=type=ssh \
    go mod download

COPY . .
RUN go build -o /app/server .
```

```bash
# Build with SSH forwarding
docker build --ssh default -t myapp:1.0 .
```

---

## Base Image Selection

Choosing the right base image affects image size, security, build speed, and debugging capability.

| Base Image | Size | Security | Debugging | Best For |
|---|---|---|---|---|
| **Alpine** (`node:20-alpine`) | ~5 MB base | Small attack surface; musl libc | `sh` available; `apk` for packages | General-purpose minimal images |
| **Distroless** (`gcr.io/distroless/static`) | ~2 MB | Minimal attack surface; no shell | No shell, no package manager | Compiled languages (Go, Rust) |
| **Chainguard** (`cgr.dev/chainguard/node`) | ~20–50 MB | CVE-free base; signed images | Limited; `apk` in `-dev` variants | Security-critical workloads |
| **Debian Slim** (`node:20-slim`) | ~80 MB | Larger surface; glibc | Full shell; `apt` for packages | Apps needing glibc or native deps |
| **Ubuntu** (`ubuntu:22.04`) | ~77 MB | Well-supported; glibc | Full shell; `apt` for packages | Apps needing latest packages |
| **scratch** | 0 MB | Zero attack surface | Nothing at all | Static binaries only |

### Decision Tree

```
Do you have a statically compiled binary?
├── Yes ──► Use scratch or distroless/static
└── No
    ├── Do you need glibc?
    │   ├── Yes ──► Use debian-slim or ubuntu
    │   └── No ──► Use alpine
    └── Is this a security-critical production image?
        ├── Yes ──► Use Chainguard or distroless
        └── No ──► Use alpine or debian-slim
```

### Alpine Considerations

Alpine uses **musl libc** instead of **glibc**. This can cause issues with:

- Pre-compiled native modules (Python wheels, Node.js native addons)
- DNS resolution behavior differences
- Performance in some CPU-intensive workloads

```dockerfile
# If you hit musl issues, switch to slim
# FROM node:20-alpine      ← musl
FROM node:20-slim           # ← glibc
```

### Distroless Example

```dockerfile
# Multi-stage with distroless runtime
FROM golang:1.22 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /server .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /server /server
ENTRYPOINT ["/server"]
```

---

## Best Practices Summary

| Practice | Description |
|---|---|
| **Use multi-stage builds** | Separate build and runtime stages to minimize image size |
| **Run as non-root** | Always set `USER` to a non-root user in production images |
| **Pin base image versions** | Use specific tags or digests, never `latest` |
| **Order layers by change frequency** | Put rarely-changing instructions first for better caching |
| **Use `.dockerignore`** | Exclude unnecessary files from the build context |
| **Combine `RUN` commands** | Merge related commands and clean up in the same layer |
| **Add `HEALTHCHECK`** | Define health checks so orchestrators know container state |
| **Use `COPY` over `ADD`** | Prefer `COPY` for explicit behavior; use `ADD` only for tar extraction |
| **Scan images for vulnerabilities** | Use `docker scout`, Trivy, or Snyk before deploying |
| **Use BuildKit** | Enable BuildKit for cache mounts, secrets, and faster parallel builds |
| **Label your images** | Use OCI standard labels for traceability and automation |
| **Minimize installed packages** | Use `--no-install-recommends` and remove caches in the same layer |

---

## Next Steps

Continue your Docker learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Docker Overview | Core Docker concepts, architecture, and CLI essentials |
| [02-CONTAINER-RUNTIME.md](02-CONTAINER-RUNTIME.md) | Container Runtime | Docker Engine, containerd, runc, and container lifecycle internals |
| [03-NETWORKING.md](03-NETWORKING.md) | Docker Networking | Bridge, host, overlay networks, DNS, and port mapping |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Docker Images and Dockerfiles documentation |
