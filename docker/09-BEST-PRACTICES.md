# Docker Best Practices

A comprehensive guide to Docker best practices for production — covering image design, Dockerfile optimization, runtime configuration, the 12-Factor App methodology, health checks, logging, tagging strategies, CI/CD integration, and operational readiness for orchestration.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Image Best Practices](#image-best-practices)
   - [Use Specific Tags](#use-specific-tags)
   - [Minimal Base Images](#minimal-base-images)
   - [Multi-Stage Builds](#multi-stage-builds)
   - [Scan Images](#scan-images)
   - [Single Process Per Container](#single-process-per-container)
3. [Dockerfile Best Practices](#dockerfile-best-practices)
   - [Order Instructions for Cache](#order-instructions-for-cache)
   - [Combine RUN Commands](#combine-run-commands)
   - [Use COPY Over ADD](#use-copy-over-add)
   - [Set WORKDIR](#set-workdir)
   - [Non-Root USER](#non-root-user)
   - [HEALTHCHECK Instruction](#healthcheck-instruction)
   - [Use .dockerignore](#use-dockerignore)
4. [Runtime Best Practices](#runtime-best-practices)
   - [Resource Limits](#resource-limits)
   - [Read-Only Root Filesystem](#read-only-root-filesystem)
   - [Restart Policies](#restart-policies)
   - [Logging Drivers](#logging-drivers)
5. [12-Factor App and Docker](#12-factor-app-and-docker)
6. [Health Checks](#health-checks)
   - [HEALTHCHECK Instruction](#healthcheck-instruction-1)
   - [HTTP Health Checks](#http-health-checks)
   - [Exec Health Checks](#exec-health-checks)
   - [Compose Health Checks](#compose-health-checks)
   - [Health Check Patterns](#health-check-patterns)
7. [Logging Best Practices](#logging-best-practices)
   - [Log to stdout and stderr](#log-to-stdout-and-stderr)
   - [Log Drivers](#log-drivers)
   - [Structured Logging](#structured-logging)
   - [Log Rotation](#log-rotation)
8. [Tagging and Versioning Strategy](#tagging-and-versioning-strategy)
   - [Semantic Versioning](#semantic-versioning)
   - [Git SHA Tags](#git-sha-tags)
   - [Never Use latest in Production](#never-use-latest-in-production)
   - [Tagging Strategy Comparison](#tagging-strategy-comparison)
9. [CI/CD Integration Patterns](#cicd-integration-patterns)
   - [Build Once Deploy Many](#build-once-deploy-many)
   - [Image Promotion](#image-promotion)
   - [Environment Parity](#environment-parity)
   - [GitHub Actions Example](#github-actions-example)
10. [Docker in Production Checklist](#docker-in-production-checklist)
11. [Security Best Practices Summary](#security-best-practices-summary)
12. [Orchestration Readiness](#orchestration-readiness)
    - [Preparing for Kubernetes](#preparing-for-kubernetes)
    - [Preparing for Docker Swarm](#preparing-for-docker-swarm)
    - [Orchestration Readiness Checklist](#orchestration-readiness-checklist)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

Running Docker in production requires more than just `docker run`. Images must be small, secure, and reproducible. Containers must be configured with resource limits, health checks, proper logging, and restart policies. CI/CD pipelines must build once and promote the same image across environments.

This document distills Docker best practices from the community, production experience, and the 12-Factor App methodology into actionable guidance for teams building and operating containerized applications.

### Target Audience

- **Developers** building and shipping containerized applications
- **DevOps Engineers** designing container build and deployment pipelines
- **SREs** operating Docker containers in production
- **Platform Engineers** defining container standards for organizations

### Scope

- Image design and Dockerfile optimization
- Runtime configuration and security
- Health checks, logging, and monitoring
- Tagging, versioning, and CI/CD integration
- Orchestration readiness for Kubernetes and Docker Swarm

---

## Image Best Practices

### Use Specific Tags

Never rely on `latest` or mutable tags for production images. Use immutable references.

```dockerfile
# ❌ Bad: Mutable tag — builds are not reproducible
FROM node:latest
FROM python:3

# ✅ Good: Specific version tag
FROM node:20.11.1-alpine3.19
FROM python:3.12.3-slim-bookworm

# ✅ Best: Pinned by digest (immutable, tamper-proof)
FROM node:20.11.1-alpine3.19@sha256:1a526b97cace5e5aae2d07fb2e8e6b7e28...
```

### Minimal Base Images

Choose the smallest base image that meets your requirements. Smaller images have fewer vulnerabilities, pull faster, and use less storage.

```
Base Image Selection Guide
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Statically compiled binary (Go, Rust)?
  ├── Yes → scratch or distroless/static
  └── No
       │
       Needs libc only?
       ├── Yes → distroless/base or alpine
       └── No
            │
            Needs system packages?
            ├── Few → alpine + apk add
            └── Many → debian-slim or ubuntu
```

| Base Image | Size | Packages | Shell | Best For |
|---|---|---|---|---|
| `scratch` | 0 MB | None | No | Static Go/Rust binaries |
| `distroless/static` | 2 MB | None | No | Static binaries needing CA certs |
| `distroless/base` | 20 MB | glibc | No | Dynamic binaries |
| `alpine` | 5 MB | musl, busybox | Yes | Most applications |
| `debian-slim` | 80 MB | apt, glibc | Yes | When alpine is incompatible |
| `ubuntu` | 78 MB | apt, glibc | Yes | Maximum compatibility |

### Multi-Stage Builds

Separate build dependencies from runtime. The final image should contain only what is needed to run the application.

```dockerfile
# Stage 1: Build
FROM golang:1.23 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /server .

# Stage 2: Runtime (only the binary)
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

```dockerfile
# Multi-stage for Node.js
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build && npm prune --production

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Scan Images

Scan every image for vulnerabilities before pushing to a registry.

```bash
# Trivy — scan and fail on CRITICAL/HIGH
trivy image --severity CRITICAL,HIGH --exit-code 1 myapp:v1.0.0

# Grype — alternative scanner
grype myapp:v1.0.0 --fail-on high

# Docker Scout (built into Docker CLI)
docker scout cves myapp:v1.0.0
```

### Single Process Per Container

Each container should run a single process. This simplifies logging, scaling, health checking, and lifecycle management.

```
✅ Single Process Per Container
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────┐  ┌─────────┐  ┌─────────┐
  │  Web    │  │  API    │  │  Worker │
  │ (nginx) │  │ (node)  │  │ (python)│
  └─────────┘  └─────────┘  └─────────┘
  Scale: 3x    Scale: 5x    Scale: 2x
  Logs: nginx  Logs: node   Logs: python

❌ Multiple Processes Per Container
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌───────────────────────────────┐
  │  nginx + node + python + cron │
  │  (supervisor manages all)     │
  └───────────────────────────────┘
  Scale: all-or-nothing
  Logs: mixed and hard to parse
  Health: which process do you check?
```

---

## Dockerfile Best Practices

### Order Instructions for Cache

Place instructions that change infrequently at the top and frequently changing instructions at the bottom.

```dockerfile
# ✅ Good: Optimized instruction order
FROM python:3.12-slim

# 1. System dependencies (rarely change)
RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# 2. Python dependencies (change sometimes)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 3. Application code (changes every commit)
COPY . .

USER nobody
CMD ["gunicorn", "-b", "0.0.0.0:8000", "app:app"]
```

### Combine RUN Commands

Each `RUN` creates a layer. Combine related commands to reduce layers and ensure cleanup happens in the same layer.

```dockerfile
# ❌ Bad: 3 layers; apt cache persists in layer 1
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ Good: 1 layer; apt cache cleaned in same layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*
```

### Use COPY Over ADD

`COPY` is explicit and predictable. `ADD` has additional features (URL downloads, tar extraction) that can cause unexpected behavior.

```dockerfile
# ❌ Avoid: ADD has implicit behavior
ADD https://example.com/file.tar.gz /app/
ADD archive.tar.gz /app/             # Auto-extracts!

# ✅ Prefer: COPY is explicit
COPY . /app/
COPY --chown=appuser:appgroup config.json /app/config.json
```

### Set WORKDIR

Use `WORKDIR` instead of `cd` in `RUN` commands. `WORKDIR` creates the directory if it does not exist and persists across instructions.

```dockerfile
# ❌ Bad: Using cd (does not persist)
RUN cd /app && npm install

# ✅ Good: Using WORKDIR
WORKDIR /app
COPY package*.json ./
RUN npm ci
```

### Non-Root USER

Always run the application as a non-root user.

```dockerfile
# Create user and switch
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --ingroup appgroup appuser

# Copy files with correct ownership
COPY --chown=appuser:appgroup . .

# Switch to non-root
USER appuser
```

### HEALTHCHECK Instruction

Define health checks in the Dockerfile so Docker can monitor container health.

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8080/healthz || exit 1
```

### Use .dockerignore

Exclude files not needed in the build context.

```
# .dockerignore
.git
.github
.gitignore
node_modules
*.md
!README.md
Dockerfile
docker-compose*.yml
.env*
tests/
coverage/
.vscode/
.idea/
__pycache__/
*.pyc
```

---

## Runtime Best Practices

### Resource Limits

Always set CPU and memory limits for production containers.

```bash
# Docker run with resource limits
docker run -d \
  --name app \
  --cpus=2 \
  --memory=512m \
  --memory-swap=512m \
  --pids-limit=100 \
  myapp:v1.0.0
```

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:v1.0.0
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
```

### Read-Only Root Filesystem

Prevent runtime modification of the container filesystem.

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --tmpfs /var/run:rw,noexec,nosuid,size=1m \
  myapp:v1.0.0
```

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:v1.0.0
    read_only: true
    tmpfs:
      - /tmp:size=64m
      - /var/run:size=1m
```

### Restart Policies

Configure restart policies to handle container failures.

| Policy | Behavior | Use Case |
|---|---|---|
| `no` | Never restart | One-off tasks, batch jobs |
| `on-failure[:max]` | Restart on non-zero exit code | Services that may crash |
| `always` | Always restart (even on stop) | Infrastructure services |
| `unless-stopped` | Restart unless manually stopped | Application services |

```bash
# Restart on failure (max 5 attempts)
docker run -d --restart=on-failure:5 myapp:v1.0.0

# Always restart (production default)
docker run -d --restart=unless-stopped myapp:v1.0.0
```

### Logging Drivers

Configure appropriate log drivers for your environment.

```bash
# Use json-file with rotation (default)
docker run -d \
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp:v1.0.0
```

```json
// /etc/docker/daemon.json — global default
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5",
    "compress": "true"
  }
}
```

---

## 12-Factor App and Docker

The [12-Factor App](https://12factor.net/) methodology maps directly to Docker patterns. Each factor aligns with a container best practice.

| Factor | Principle | Docker Implementation |
|---|---|---|
| **I. Codebase** | One codebase, many deploys | One Dockerfile per service; same image across environments |
| **II. Dependencies** | Explicitly declare dependencies | `requirements.txt`, `package.json`, `go.mod` in image |
| **III. Config** | Store config in the environment | `ENV`, `--env`, `--env-file`, ConfigMaps |
| **IV. Backing Services** | Treat backing services as attached resources | Connect via environment variables (`DATABASE_URL`) |
| **V. Build, Release, Run** | Strictly separate build and run | Multi-stage build → push to registry → deploy image |
| **VI. Processes** | Execute as stateless processes | No local state; use volumes or external stores |
| **VII. Port Binding** | Export services via port binding | `EXPOSE` + `-p` flag |
| **VIII. Concurrency** | Scale out via process model | Scale with `docker compose up --scale app=5` |
| **IX. Disposability** | Fast startup and graceful shutdown | Handle SIGTERM; use `--init` for signal propagation |
| **X. Dev/Prod Parity** | Keep environments similar | Same image in dev, staging, production |
| **XI. Logs** | Treat logs as event streams | Log to stdout/stderr; use log drivers |
| **XII. Admin Processes** | Run admin tasks as one-off processes | `docker exec` or `docker run --rm` for migrations |

```dockerfile
# 12-Factor Docker Example
FROM python:3.12-slim

# Factor II: Explicit dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

WORKDIR /app
COPY . .

# Factor III: Config from environment
ENV APP_PORT=8000
ENV LOG_LEVEL=info

# Factor VII: Port binding
EXPOSE ${APP_PORT}

# Factor IX: Graceful shutdown
STOPSIGNAL SIGTERM

# Factor XI: Logs to stdout
USER nobody
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--access-logfile", "-", "app:app"]
```

---

## Health Checks

### HEALTHCHECK Instruction

Docker supports health checks that report whether a container is functioning correctly. Docker marks the container as `healthy`, `unhealthy`, or `starting` based on the check result.

```
Health Check State Machine
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Container Start
       │
       ▼
  ┌──────────┐    start-period elapsed    ┌──────────┐
  │ starting │ ──────────────────────────▶ │ healthy  │
  └──────────┘                             └────┬─────┘
                                                │
                              retries failures  │
                                                ▼
                                          ┌───────────┐
                                          │ unhealthy │
                                          └───────────┘
```

| Parameter | Description | Default | Recommendation |
|---|---|---|---|
| `--interval` | Time between checks | 30s | 10-30s |
| `--timeout` | Max time for check to complete | 30s | 3-5s |
| `--start-period` | Grace period for startup | 0s | Set to expected startup time |
| `--retries` | Failures before marking unhealthy | 3 | 3-5 |

### HTTP Health Checks

```dockerfile
# HTTP health check with curl
HEALTHCHECK --interval=15s --timeout=3s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8080/healthz || exit 1
```

```dockerfile
# HTTP health check without curl (wget, available in alpine)
HEALTHCHECK --interval=15s --timeout=3s --start-period=30s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/healthz || exit 1
```

### Exec Health Checks

For non-HTTP services (databases, message queues, workers).

```dockerfile
# PostgreSQL health check
HEALTHCHECK --interval=10s --timeout=5s --retries=5 \
  CMD pg_isready -U postgres || exit 1

# Redis health check
HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
  CMD redis-cli ping | grep -q PONG || exit 1

# Worker process health check (check PID file)
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD test -f /tmp/worker.pid && kill -0 $(cat /tmp/worker.pid) || exit 1
```

### Compose Health Checks

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  app:
    image: myapp:v1.0.0
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/healthz"]
      interval: 15s
      timeout: 3s
      retries: 3
      start_period: 20s

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
```

### Health Check Patterns

| Pattern | Check Type | What It Validates |
|---|---|---|
| **Liveness** | Lightweight exec or HTTP | Process is alive and not deadlocked |
| **Readiness** | HTTP with dependency checks | Can accept and process requests |
| **Startup** | HTTP or exec with long timeout | Application has finished initialization |
| **Deep health** | HTTP returning JSON status | All dependencies operational (for dashboards, not probes) |

---

## Logging Best Practices

### Log to stdout and stderr

Docker captures stdout and stderr from the container's PID 1 process. Do not write logs to files inside the container.

```dockerfile
# ✅ Good: Configure application to log to stdout
CMD ["node", "server.js"]
# (application uses console.log which writes to stdout)

# ✅ Good: Redirect Nginx logs to stdout/stderr
RUN ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log
```

```bash
# View container logs
docker logs myapp
docker logs --tail 100 --follow myapp
docker logs --since 1h myapp
```

### Log Drivers

| Driver | Description | Pros | Cons |
|---|---|---|---|
| `json-file` | JSON file on host (default) | Simple, `docker logs` works | No centralization |
| `local` | Optimized local storage | Faster than json-file | `docker logs` works |
| `syslog` | Send to syslog server | Standard protocol | Older format |
| `journald` | Send to systemd journal | systemd integration | Linux only |
| `fluentd` | Send to Fluentd | Powerful routing | Requires Fluentd |
| `awslogs` | Send to CloudWatch | AWS native | AWS only |
| `gcplogs` | Send to Cloud Logging | GCP native | GCP only |

### Structured Logging

Use structured (JSON) logging for machine-parseable log entries.

```python
# Python — structured logging with structlog
import structlog

logger = structlog.get_logger()

logger.info(
    "request_processed",
    method="GET",
    path="/api/users",
    status=200,
    duration_ms=45,
    user_id="user-123",
)
# Output: {"event": "request_processed", "method": "GET", "path": "/api/users", ...}
```

```javascript
// Node.js — structured logging with pino
const pino = require('pino');
const logger = pino({ level: 'info' });

logger.info({
  msg: 'request processed',
  method: 'GET',
  path: '/api/users',
  statusCode: 200,
  durationMs: 45,
  userId: 'user-123',
});
```

### Log Rotation

Prevent containers from filling the host disk with logs.

```json
// /etc/docker/daemon.json — global log rotation
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5",
    "compress": "true"
  }
}
```

```bash
# Per-container log rotation
docker run -d \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp:v1.0.0
```

> **Warning:** Without log rotation, a single verbose container can fill the host disk, crashing all containers on that host. Always configure `max-size` and `max-file`.

---

## Tagging and Versioning Strategy

### Semantic Versioning

Tag images with semantic versions for clear release tracking.

```bash
# Tag with semantic version
docker tag myapp:latest myregistry.io/myapp:1.0.0
docker tag myapp:latest myregistry.io/myapp:1.0
docker tag myapp:latest myregistry.io/myapp:1

# Push all tags
docker push myregistry.io/myapp:1.0.0
docker push myregistry.io/myapp:1.0
docker push myregistry.io/myapp:1
```

### Git SHA Tags

Use the Git commit SHA for full traceability from image to source code.

```bash
# Tag with Git SHA
GIT_SHA=$(git rev-parse --short HEAD)
docker tag myapp:latest myregistry.io/myapp:${GIT_SHA}

# Tag with both semver and Git SHA
docker tag myapp:latest myregistry.io/myapp:1.0.0
docker tag myapp:latest myregistry.io/myapp:1.0.0-${GIT_SHA}
```

### Never Use latest in Production

The `latest` tag is mutable — it points to whatever was last pushed. Using it in production means deployments are not reproducible and rollbacks are impossible.

```yaml
# ❌ Bad: Non-reproducible deployment
services:
  app:
    image: myapp:latest

# ✅ Good: Reproducible and traceable
services:
  app:
    image: myregistry.io/myapp:1.0.0@sha256:abc123...
```

### Tagging Strategy Comparison

| Strategy | Example | Traceability | Rollback | Use Case |
|---|---|---|---|---|
| **Semver** | `v1.2.3` | Version history | ✅ Easy | Releases |
| **Git SHA** | `abc1234` | Exact commit | ✅ Easy | CI/CD builds |
| **Semver + SHA** | `v1.2.3-abc1234` | Both | ✅ Easy | Best of both |
| **Date** | `2025-01-15` | Date-based | 🟡 OK | Nightly builds |
| **Branch** | `main`, `develop` | Branch only | ❌ Hard | Dev environments only |
| **latest** | `latest` | ❌ None | ❌ None | ❌ Never in production |

---

## CI/CD Integration Patterns

### Build Once Deploy Many

Build the image once and promote the same artifact across environments. Never rebuild for staging or production.

```
Build Once, Deploy Many
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Source Code
      │
      ▼
  ┌──────────┐      ┌──────────┐
  │  Build   │─────▶│ Registry │
  │  Image   │      │ (store)  │
  │ (once)   │      └────┬─────┘
  └──────────┘           │
                         │ same image SHA
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         ┌────────┐ ┌────────┐ ┌────────┐
         │  Dev   │ │Staging │ │  Prod  │
         │ env    │ │ env    │ │ env    │
         └────────┘ └────────┘ └────────┘

  Configuration differences via environment variables,
  NOT different images.
```

### Image Promotion

Promote images between environments using tags, not rebuilds.

```bash
# After testing passes in staging, promote to production
# Option 1: Re-tag and push
docker pull myregistry.io/myapp:staging-abc1234
docker tag myregistry.io/myapp:staging-abc1234 myregistry.io/myapp:prod-abc1234
docker push myregistry.io/myapp:prod-abc1234

# Option 2: Use the same digest across environments
# staging and production reference the same SHA
docker pull myregistry.io/myapp@sha256:abc123...
```

### Environment Parity

Keep development, staging, and production environments as similar as possible.

```yaml
# docker-compose.yml — base configuration
services:
  app:
    image: myregistry.io/myapp:${IMAGE_TAG}
    env_file:
      - .env.${ENVIRONMENT}
    deploy:
      resources:
        limits:
          cpus: "${CPU_LIMIT}"
          memory: "${MEMORY_LIMIT}"
```

```bash
# .env.development
IMAGE_TAG=latest
CPU_LIMIT=1
MEMORY_LIMIT=256M

# .env.production
IMAGE_TAG=1.0.0
CPU_LIMIT=4
MEMORY_LIMIT=2G
```

### GitHub Actions Example

```yaml
# .github/workflows/docker-build.yml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags: ["v*"]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
```

---

## Docker in Production Checklist

| Category | Check | Priority | Status |
|---|---|---|---|
| **Image** | Use specific, pinned base image tags | 🔴 Critical | ☐ |
| **Image** | Multi-stage builds for minimal runtime image | 🟠 High | ☐ |
| **Image** | Scan images for vulnerabilities in CI/CD | 🔴 Critical | ☐ |
| **Image** | Sign images with cosign or DCT | 🟠 High | ☐ |
| **Image** | Single process per container | 🟠 High | ☐ |
| **Dockerfile** | Run as non-root USER | 🔴 Critical | ☐ |
| **Dockerfile** | Order instructions for cache optimization | 🟡 Medium | ☐ |
| **Dockerfile** | Use COPY, not ADD | 🟡 Medium | ☐ |
| **Dockerfile** | No secrets in image layers | 🔴 Critical | ☐ |
| **Dockerfile** | .dockerignore configured | 🟡 Medium | ☐ |
| **Dockerfile** | HEALTHCHECK defined | 🟠 High | ☐ |
| **Runtime** | CPU limits set (`--cpus`) | 🔴 Critical | ☐ |
| **Runtime** | Memory limits set (`--memory`) | 🔴 Critical | ☐ |
| **Runtime** | Read-only root filesystem (`--read-only`) | 🟠 High | ☐ |
| **Runtime** | Restart policy configured | 🟠 High | ☐ |
| **Runtime** | PID limits set (`--pids-limit`) | 🟡 Medium | ☐ |
| **Logging** | Application logs to stdout/stderr | 🔴 Critical | ☐ |
| **Logging** | Log rotation configured (`max-size`, `max-file`) | 🔴 Critical | ☐ |
| **Logging** | Structured (JSON) logging | 🟠 High | ☐ |
| **Networking** | Containers on isolated networks | 🟠 High | ☐ |
| **Networking** | Docker socket not exposed | 🔴 Critical | ☐ |
| **Security** | Capabilities dropped (`--cap-drop=ALL`) | 🟠 High | ☐ |
| **Security** | Seccomp profile applied | 🟡 Medium | ☐ |
| **Tags** | Semver or Git SHA tags (never `latest`) | 🔴 Critical | ☐ |
| **Tags** | Images pinned by digest in production | 🟠 High | ☐ |
| **CI/CD** | Build once, deploy to all environments | 🟠 High | ☐ |
| **CI/CD** | Image scanning in pipeline | 🔴 Critical | ☐ |
| **Orchestration** | Graceful shutdown (SIGTERM handling) | 🟠 High | ☐ |
| **Orchestration** | Health checks compatible with orchestrator | 🟠 High | ☐ |

---

## Security Best Practices Summary

For comprehensive Docker security guidance, see [07-SECURITY.md](07-SECURITY.md). Key highlights:

- **Never run as root** — always specify `USER` in your Dockerfile
- **Never use `--privileged`** — drop all capabilities and add only required ones
- **Never expose the Docker socket** — it grants full host access
- **Never embed secrets in images** — use BuildKit `--secret` or external vaults
- **Scan images** — integrate Trivy, Grype, or Snyk into CI/CD
- **Sign images** — use cosign for supply chain integrity
- **Use read-only filesystems** — prevent runtime modifications
- **Segment networks** — isolate tiers with separate Docker networks
- **Keep images minimal** — smaller images have fewer vulnerabilities

```
Security Defense in Depth
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────────────────────────────────┐
  │  Supply Chain: signed images, SBOMs     │
  ├─────────────────────────────────────────┤
  │  Network: segmentation, no socket       │
  ├─────────────────────────────────────────┤
  │  Runtime: caps, seccomp, read-only FS   │
  ├─────────────────────────────────────────┤
  │  Image: minimal base, no secrets, scan  │
  ├─────────────────────────────────────────┤
  │  Host: patched kernel, rootless, userns │
  └─────────────────────────────────────────┘
```

---

## Orchestration Readiness

### Preparing for Kubernetes

Containers destined for Kubernetes need specific properties to work well in a cluster environment.

```yaml
# Kubernetes-ready container properties
# 1. Health checks (maps to liveness/readiness probes)
# 2. Graceful shutdown (SIGTERM handling)
# 3. Stateless (external state via PVCs/services)
# 4. Config via environment variables
# 5. Logs to stdout/stderr

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myregistry.io/myapp:1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          env:
            - name: LOG_LEVEL
              value: info
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
```

### Preparing for Docker Swarm

```yaml
# docker-compose.yml for Swarm deployment
services:
  app:
    image: myregistry.io/myapp:1.0.0
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      rollback_config:
        parallelism: 1
        delay: 5s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/healthz"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s
    networks:
      - frontend
    secrets:
      - db_password

secrets:
  db_password:
    external: true

networks:
  frontend:
    driver: overlay
```

### Orchestration Readiness Checklist

| Requirement | Docker Run | Kubernetes | Swarm |
|---|---|---|---|
| Health checks | `HEALTHCHECK` in Dockerfile | `livenessProbe` / `readinessProbe` | `healthcheck` in Compose |
| Graceful shutdown | Handle SIGTERM, use `--init` | `terminationGracePeriodSeconds` | `stop_grace_period` |
| Resource limits | `--cpus`, `--memory` | `resources.limits` | `deploy.resources.limits` |
| Non-root user | `USER` in Dockerfile | `runAsNonRoot: true` | `USER` in Dockerfile |
| Stateless | External volumes | PVCs, ConfigMaps, Secrets | Volumes, Secrets |
| Logging | stdout/stderr | stdout/stderr + log aggregation | stdout/stderr + log driver |
| Config | env vars, files | ConfigMaps, Secrets, env vars | env vars, Configs, Secrets |
| Networking | Docker networks | Services, Ingress, NetworkPolicy | Overlay networks |
| Scaling | `docker compose --scale` | HPA, replicas | `deploy.replicas` |
| Rolling updates | Manual | `strategy.rollingUpdate` | `update_config` |

---

## Next Steps

Continue your Docker learning journey:

| File | Topic | Description |
|---|---|---|
| [07-SECURITY.md](07-SECURITY.md) | Docker Security | Container isolation, image hardening, runtime security, supply chain |
| [08-PERFORMANCE.md](08-PERFORMANCE.md) | Docker Performance | Resource limits, image optimization, build performance, monitoring |
| [05-DOCKER-COMPOSE.md](05-DOCKER-COMPOSE.md) | Docker Compose | Multi-container applications with Compose files |
| [06-REGISTRIES.md](06-REGISTRIES.md) | Container Registries | Registry services, image signing, and authentication |
| [01-IMAGES-AND-DOCKERFILES.md](01-IMAGES-AND-DOCKERFILES.md) | Images & Dockerfiles | Multi-stage builds, Dockerfile instructions, and image layering |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Docker Best Practices documentation |
