# Docker Anti-Patterns

A catalogue of the most common Docker and container mistakes — what they look like, why they are harmful, and exactly how to fix them. Use this document as a code-review checklist, a pre-production gate, or a team learning resource.

---

## Table of Contents

- [Introduction](#introduction)
- [Anti-Patterns Summary Table](#anti-patterns-summary-table)
- [1. Running as Root](#1-running-as-root)
- [2. Using :latest Tag in Production](#2-using-latest-tag-in-production)
- [3. Storing Secrets in Images](#3-storing-secrets-in-images)
- [4. Fat Images](#4-fat-images)
- [5. One Container, Many Processes](#5-one-container-many-processes)
- [6. Not Using .dockerignore](#6-not-using-dockerignore)
- [7. Installing Unnecessary Packages](#7-installing-unnecessary-packages)
- [8. Not Setting Resource Limits](#8-not-setting-resource-limits)
- [9. Ignoring Health Checks](#9-ignoring-health-checks)
- [10. Modifying Running Containers](#10-modifying-running-containers)
- [11. Hardcoding Configuration](#11-hardcoding-configuration)
- [12. Not Cleaning Up](#12-not-cleaning-up)
- [Quick Reference Checklist](#quick-reference-checklist)
- [Next Steps](#next-steps)
- [Version History](#version-history)

---

## Introduction

### Why Anti-Patterns Matter

Anti-patterns are recurring practices that seem reasonable at first glance but create significant problems over time. In Docker and container workflows, the consequences of anti-patterns are typically felt during security incidents, production outages, or CI/CD pipeline failures — the exact moments when your container infrastructure needs to work flawlessly.

The patterns documented here represent real failures observed across production container environments. Each one is:

- **Seductive** — it felt like the right approach when first implemented
- **Harmful** — it creates security vulnerabilities, bloated images, unreliable deployments, or operational overhead
- **Fixable** — there is a well-understood better approach

### How to Use This Document

1. **Pre-production review:** Use the [Quick Reference Checklist](#quick-reference-checklist) before deploying a new containerised service
2. **Code review:** Reference specific sections when reviewing Dockerfile or Compose PRs
3. **Incident retros:** After an incident, check which anti-patterns contributed to the failure
4. **Team onboarding:** Assign this document to new engineers before they write production Dockerfiles
5. **Periodic audit:** Run through the checklist quarterly for existing images and Compose stacks

---

## Anti-Patterns Summary Table

| # | Anti-Pattern | Category | Severity |
|---|--------------|----------|----------|
| 1 | Running as Root | Security | 🔴 Critical |
| 2 | Using :latest Tag in Production | Reliability | 🔴 Critical |
| 3 | Storing Secrets in Images | Security | 🔴 Critical |
| 4 | Fat Images | Build / Size | 🟠 High |
| 5 | One Container, Many Processes | Architecture | 🟠 High |
| 6 | Not Using .dockerignore | Build / Security | 🟠 High |
| 7 | Installing Unnecessary Packages | Size / Security | 🟠 High |
| 8 | Not Setting Resource Limits | Reliability | 🟠 High |
| 9 | Ignoring Health Checks | Reliability | 🟡 Medium |
| 10 | Modifying Running Containers | Operations | 🟡 Medium |
| 11 | Hardcoding Configuration | Portability | 🟡 Medium |
| 12 | Not Cleaning Up | Operations | 🟡 Medium |

---

## 1. Running as Root

### Problem

By default, Docker containers run their main process as `root` (UID 0). If an attacker exploits a vulnerability in the application, they gain root-level access inside the container. Combined with a container escape vulnerability or a misconfigured volume mount, this can escalate to root access on the host machine.

### Example of the Bad Pattern

```dockerfile
# Bad: no USER instruction — process runs as root
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

Running this container and checking the user:

```bash
$ docker exec my-app whoami
root

$ docker exec my-app id
uid=0(root) gid=0(root) groups=0(root)
```

**Consequences:**

- Application process has unrestricted filesystem access inside the container
- If a remote code execution (RCE) vulnerability is exploited, the attacker is root
- Host volumes mounted into the container can be modified by root
- Container escape CVEs (e.g., CVE-2019-5736 in runc) are far more dangerous when the container process is root

### Why It's Harmful

- Root in the container maps to root on the host in many default configurations
- Attackers can install tools, modify binaries, and pivot to other containers
- Compliance frameworks (SOC 2, PCI-DSS, CIS Benchmarks) flag root containers
- A non-root container limits the blast radius of any exploit to an unprivileged user

### Recommended Fix

```dockerfile
# Good: create a non-root user and switch to it
FROM node:20-slim

# Create a dedicated application user
RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --shell /bin/false --create-home appuser

WORKDIR /app
COPY package*.json ./
RUN npm ci --production

# Change ownership and switch user BEFORE copying application code
COPY --chown=appuser:appgroup . .

USER appuser

EXPOSE 3000
CMD ["node", "server.js"]
```

Verify:

```bash
$ docker exec my-app whoami
appuser

$ docker exec my-app id
uid=1001(appuser) gid=1001(appgroup) groups=1001(appgroup)
```

For even stronger isolation, use `--read-only` and `--cap-drop ALL` at runtime:

```bash
docker run --user 1001:1001 \
  --read-only \
  --cap-drop ALL \
  --tmpfs /tmp \
  my-app:1.0.0
```

### Quick Check

- [ ] Every Dockerfile has a `USER` instruction that switches to a non-root user
- [ ] The non-root user has no login shell (`/bin/false` or `/usr/sbin/nologin`)
- [ ] `docker exec <container> id` confirms UID ≠ 0
- [ ] Runtime uses `--cap-drop ALL` and adds back only required capabilities
- [ ] CI pipeline runs a CIS benchmark scanner (e.g., `dockle`, `docker scout`)

---

## 2. Using :latest Tag in Production

### Problem

Using the `:latest` tag (or no tag at all, which defaults to `:latest`) for production deployments means you have no control over which version of an image is running. Builds are non-reproducible, rollbacks are impossible, and two identical `docker pull` commands minutes apart may yield entirely different images.

### Example of the Bad Pattern

```yaml
# docker-compose.prod.yml — using :latest everywhere
services:
  api:
    image: mycompany/api:latest
  worker:
    image: mycompany/worker:latest
  redis:
    image: redis:latest
  postgres:
    image: postgres:latest
```

```dockerfile
# Dockerfile — base image uses :latest
FROM python:latest
COPY . /app
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

**What happens:**

```
Monday:   docker pull redis:latest → Redis 7.2.3
Tuesday:  docker pull redis:latest → Redis 7.2.4 (minor bug fix)
Thursday: docker pull redis:latest → Redis 7.4.0 (breaking change!)

Your staging environment runs 7.2.3.
Your production environment pulls 7.4.0.
The deploy "passes" staging but breaks production.
```

### Why It's Harmful

- Builds are not reproducible — the same Dockerfile produces different images on different days
- Cannot roll back to a known-good version if `:latest` has been overwritten
- Staging and production may run different image versions without anyone knowing
- Debugging is impossible if you do not know which version introduced a bug
- Container orchestrators (Kubernetes, ECS) may pull a new `:latest` on node reschedule

### Recommended Fix

```yaml
# docker-compose.prod.yml — pinned to immutable digests or SemVer tags
services:
  api:
    image: mycompany/api:1.4.2
  worker:
    image: mycompany/worker:1.4.2
  redis:
    image: redis:7.2.3-alpine
  postgres:
    image: postgres:16.1-alpine
```

```dockerfile
# Dockerfile — base image pinned to specific version
FROM python:3.12.1-slim-bookworm

COPY requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r /app/requirements.txt

COPY . /app
WORKDIR /app
CMD ["python", "app.py"]
```

For maximum reproducibility, pin to the image digest:

```yaml
# Immutable — even if the tag is moved, the digest never changes
image: redis:7.2.3-alpine@sha256:6c42cce2871e8dc5fb3e843ed5c1e4c99a10c5349f2a02e34dce31b2fcdd30d4
```

**Tagging strategy in CI/CD:**

```bash
# Build and tag with Git SHA + SemVer
IMAGE_TAG="${SEMVER_TAG}-${GIT_SHA:0:8}"
docker build -t mycompany/api:${IMAGE_TAG} .
docker tag mycompany/api:${IMAGE_TAG} mycompany/api:${SEMVER_TAG}
docker push mycompany/api:${IMAGE_TAG}
docker push mycompany/api:${SEMVER_TAG}
```

### Quick Check

- [ ] No Dockerfile uses `FROM <image>:latest` or `FROM <image>` without a tag
- [ ] No Compose file or Kubernetes manifest uses `:latest` in production
- [ ] Base images are pinned to specific version tags (e.g., `python:3.12.1-slim-bookworm`)
- [ ] CI/CD pipeline tags images with SemVer + Git SHA
- [ ] Image digests are used for critical infrastructure dependencies

---

## 3. Storing Secrets in Images

### Problem

Embedding secrets — API keys, database passwords, TLS certificates, cloud credentials — directly in the Dockerfile, environment variables baked into the image, or copied files means that anyone who can pull the image can extract those secrets. Docker image layers are permanent and inspectable; even if you delete a secret in a later layer, it remains in the image history.

### Example of the Bad Pattern

```dockerfile
# Bad: secrets baked into the image
FROM python:3.12-slim

# Secret in ENV — visible in docker inspect and image history
ENV DATABASE_URL=postgres://admin:s3cretP@ss@prod-db:5432/myapp
ENV AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
ENV AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

COPY . /app
WORKDIR /app

# Secret in a file — still present in image layers
COPY .env /app/.env
COPY credentials.json /app/credentials.json

RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

Extracting secrets from the image:

```bash
# Anyone with image access can see ENV values
$ docker inspect myapp:1.0 --format='{{json .Config.Env}}' | jq .
[
  "DATABASE_URL=postgres://admin:s3cretP@ss@prod-db:5432/myapp",
  "AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE",
  "AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
]

# History reveals COPY commands with secret files
$ docker history myapp:1.0
IMAGE          CREATED        CREATED BY                               SIZE
a1b2c3d4e5f6   2 hours ago   COPY .env /app/.env                      512B
```

### Why It's Harmful

- Image layers are immutable — deleting a file in a later layer does not remove it from history
- Anyone with `docker pull` access can extract every secret
- Registry compromises expose all secrets in all images
- Secrets in version-controlled Dockerfiles appear in Git history forever
- Violates the principle of least privilege and every compliance standard

### Recommended Fix

```dockerfile
# Good: no secrets in the image at all
FROM python:3.12-slim

COPY requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r /app/requirements.txt

COPY . /app
WORKDIR /app

# Application reads secrets from environment variables at runtime
CMD ["python", "app.py"]
```

Pass secrets at runtime, never at build time:

```yaml
# docker-compose.yml — secrets via environment or Docker secrets
services:
  api:
    image: mycompany/api:1.4.2
    environment:
      - DATABASE_URL  # value from host env or .env file (not in image)
    secrets:
      - db_password
      - api_key

secrets:
  db_password:
    external: true   # managed by Docker Swarm or external secret manager
  api_key:
    external: true
```

If you need secrets during the build (e.g., to install private packages), use BuildKit secret mounts:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim

# Secret is mounted at build time but NEVER stored in a layer
RUN --mount=type=secret,id=pip_index_url \
    PIP_INDEX_URL=$(cat /run/secrets/pip_index_url) \
    pip install --no-cache-dir -r requirements.txt
```

```bash
# Pass the secret at build time — not persisted in the image
docker build --secret id=pip_index_url,src=.pip_index_url .
```

### Quick Check

- [ ] `docker inspect <image> --format='{{json .Config.Env}}'` shows no secrets
- [ ] `docker history <image>` shows no COPY of credential files
- [ ] Dockerfile contains no `ENV` lines with passwords, tokens, or API keys
- [ ] BuildKit `--mount=type=secret` used for any build-time secrets
- [ ] CI pipeline runs a secret scanner (e.g., `trivy`, `trufflehog`, `docker scout`)

---

## 4. Fat Images

### Problem

Building production images without multi-stage builds results in images that contain compilers, build tools, source code, test files, and development dependencies — none of which are needed at runtime. These "fat" images are 5–20× larger than necessary, take longer to pull, consume more registry storage, and have a larger attack surface.

### Example of the Bad Pattern

```dockerfile
# Bad: single-stage build — everything in one image
FROM golang:1.22

WORKDIR /app
COPY . .

# Build tools, source code, test files all remain in the final image
RUN go mod download
RUN go build -o /app/server .

EXPOSE 8080
CMD ["/app/server"]
```

```bash
$ docker images myapp
REPOSITORY   TAG     SIZE
myapp        bad     1.2GB    # Go toolchain (800MB) + source + deps
```

**What's in that 1.2 GB image:**

```
Go compiler and toolchain     ~800 MB
Source code                    ~50 MB
Build cache and module cache   ~300 MB
Compiled binary                ~15 MB   ← the only thing needed at runtime
Test files and fixtures        ~35 MB
```

### Why It's Harmful

- Larger images mean slower pulls, deploys, and autoscaling
- Build tools and compilers increase attack surface (CVE exposure)
- Source code leakage — anyone with image access can read your code
- Registry storage costs scale linearly with image size
- Cold-start times in serverless or auto-scaling environments increase dramatically
- More packages = more vulnerabilities reported by scanners = alert fatigue

### Recommended Fix

```dockerfile
# Good: multi-stage build — minimal final image
# Stage 1: Build
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server .

# Stage 2: Runtime — only the compiled binary
FROM gcr.io/distroless/static-debian12:nonroot

COPY --from=builder /app/server /server

USER nonroot:nonroot
EXPOSE 8080
CMD ["/server"]
```

```bash
$ docker images myapp
REPOSITORY   TAG      SIZE
myapp        good     12MB    # Just the binary + minimal OS
myapp        bad      1.2GB   # 100× larger
```

**Multi-stage build for Node.js:**

```dockerfile
# Stage 1: Install dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --production

# Stage 2: Minimal runtime
FROM node:20-alpine
WORKDIR /app

RUN addgroup -g 1001 appgroup && adduser -u 1001 -G appgroup -s /bin/false -D appuser

COPY --from=deps /app/node_modules ./node_modules
COPY --chown=appuser:appgroup . .

USER appuser
EXPOSE 3000
CMD ["node", "server.js"]
```

**Image size comparison:**

```
┌──────────────────────────────────────────────────────────┐
│              Image Size Comparison                        │
├──────────────────────┬───────────┬───────────────────────┤
│ Approach             │ Size      │ Notes                 │
├──────────────────────┼───────────┼───────────────────────┤
│ golang:1.22          │ 1.2 GB    │ Full toolchain        │
│ Multi-stage + alpine │ 25 MB     │ Alpine + binary       │
│ Multi-stage + distro │ 12 MB     │ Distroless + binary   │
│ Multi-stage + scratch│ 8 MB      │ Binary only           │
├──────────────────────┼───────────┼───────────────────────┤
│ node:20              │ 1.1 GB    │ Full Debian + Node    │
│ node:20-slim         │ 250 MB    │ Slim Debian + Node    │
│ node:20-alpine       │ 180 MB    │ Alpine + Node         │
└──────────────────────┴───────────┴───────────────────────┘
```

### Quick Check

- [ ] Every production Dockerfile uses a multi-stage build
- [ ] Final stage uses `alpine`, `slim`, `distroless`, or `scratch` as base
- [ ] No compilers, build tools, or source code in the final image
- [ ] `docker images` shows production images < 200 MB (language dependent)
- [ ] `docker scout` or `trivy` reports fewer CVEs in the minimal image

---

## 5. One Container, Many Processes

### Problem

Running multiple unrelated processes (e.g., web server, database, background worker, cron) inside a single container violates the single-responsibility principle. Docker's process model assumes PID 1 is the main application — when multiple processes run together, lifecycle management, logging, scaling, and health checking all break down.

### Example of the Bad Pattern

```dockerfile
# Bad: running everything in one container
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y \
    nginx \
    python3 \
    python3-pip \
    cron \
    supervisor \
    redis-server \
    postgresql

COPY supervisord.conf /etc/supervisor/conf.d/
COPY nginx.conf /etc/nginx/nginx.conf
COPY app/ /app/
COPY crontab /etc/cron.d/my-cron

RUN pip3 install -r /app/requirements.txt

# supervisord manages nginx, app, cron, redis, and postgres
CMD ["/usr/bin/supervisord"]
```

```ini
# supervisord.conf — kitchen sink
[program:nginx]
command=/usr/sbin/nginx -g "daemon off;"

[program:app]
command=python3 /app/main.py

[program:cron]
command=/usr/sbin/cron -f

[program:redis]
command=redis-server

[program:postgres]
command=/usr/lib/postgresql/14/bin/postgres -D /var/lib/postgresql/14/main
```

### Why It's Harmful

- Cannot scale individual components independently (need more workers but not more databases?)
- Logging from all processes is interleaved in a single stream — parsing and filtering becomes a nightmare
- If one process crashes, the others may continue running in a degraded state without detection
- Docker health checks can only check one process — a healthy nginx does not mean a healthy worker
- Resource limits apply to the entire container, not individual processes
- Update one component and you must rebuild and redeploy everything

### Recommended Fix

```yaml
# Good: one process per container, orchestrated with Compose
services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      api:
        condition: service_healthy

  api:
    build: ./api
    expose:
      - "8000"
    environment:
      - REDIS_URL=redis://redis:6379
      - DATABASE_URL=postgres://postgres:5432/myapp
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/healthz"]
      interval: 10s
      timeout: 5s
      retries: 3

  worker:
    build: ./worker
    environment:
      - REDIS_URL=redis://redis:6379
    deploy:
      replicas: 3    # scale workers independently

  redis:
    image: redis:7.2-alpine
    volumes:
      - redis-data:/data

  postgres:
    image: postgres:16-alpine
    volumes:
      - pg-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=myapp

volumes:
  redis-data:
  pg-data:
```

### Quick Check

- [ ] Each container runs exactly one main process
- [ ] `docker top <container>` shows only one application process (plus init if used)
- [ ] No `supervisord`, `systemd`, or custom process managers inside containers
- [ ] Each service can be scaled independently with `docker compose up --scale`
- [ ] Logs from each container contain output from only one application

---

## 6. Not Using .dockerignore

### Problem

Without a `.dockerignore` file, `docker build` sends the entire build context directory to the Docker daemon. This includes `.git/` (which can be hundreds of megabytes), `node_modules/`, test data, IDE configuration, local `.env` files with secrets, and any other file in the project root.

### Example of the Bad Pattern

```
# Project directory — no .dockerignore
my-project/
├── .git/              ← 200 MB of Git history
├── .env               ← DATABASE_URL=postgres://admin:password@...
├── .env.production    ← production secrets
├── node_modules/      ← 500 MB (will be reinstalled in build anyway)
├── coverage/          ← test coverage reports
├── .nyc_output/       ← test artifacts
├── dist/              ← old build artifacts
├── test/              ← test files not needed in production
├── docker-compose.yml
├── Dockerfile
├── package.json
└── src/
```

```bash
$ docker build -t myapp .
Sending build context to Docker daemon  847.3MB    # ← should be < 5 MB
```

### Why It's Harmful

- Build context transfer takes minutes instead of seconds
- `.env` files with secrets are sent to the daemon and may end up in image layers
- `.git/` directory exposes full repository history, credentials, and author information
- `node_modules/` or `vendor/` directories are reinstalled during the build — sending them is pure waste
- Layer cache is invalidated by irrelevant file changes (editing a test file triggers a full rebuild)
- CI/CD build times increase significantly

### Recommended Fix

```gitignore
# .dockerignore — comprehensive
# Version control
.git
.gitignore

# Dependencies (reinstalled during build)
node_modules
vendor
__pycache__
*.pyc
.venv

# IDE and editor
.vscode
.idea
*.swp
*.swo

# Environment and secrets
.env
.env.*
*.pem
*.key

# Test and coverage
test
tests
coverage
.nyc_output
*.test.js
*.spec.js

# Build artifacts
dist
build

# Documentation
docs
*.md
!README.md

# Docker (avoid recursive context)
docker-compose*.yml
Dockerfile*
.dockerignore

# OS files
.DS_Store
Thumbs.db
```

```bash
# After adding .dockerignore:
$ docker build -t myapp .
Sending build context to Docker daemon  4.2MB    # ← 200× smaller
```

### Quick Check

- [ ] Every project with a Dockerfile has a `.dockerignore` file
- [ ] `.git/`, `node_modules/`, `__pycache__/`, and `vendor/` are excluded
- [ ] `.env` and all secret files are excluded
- [ ] Build context size is logged in CI — alert if it exceeds 50 MB
- [ ] `docker build` output shows a context size under 50 MB

---

## 7. Installing Unnecessary Packages

### Problem

Installing packages "just in case" — build tools, debugging utilities, documentation, recommended packages, or entire desktop environments — bloats the image with software that is never used in production. Each unnecessary package adds to the image size and introduces potential CVEs.

### Example of the Bad Pattern

```dockerfile
# Bad: installing everything including the kitchen sink
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y \
    python3 \
    python3-pip \
    vim \
    nano \
    curl \
    wget \
    htop \
    strace \
    net-tools \
    iputils-ping \
    dnsutils \
    telnet \
    gcc \
    g++ \
    make \
    git \
    ssh \
    man-db \
    less \
    && rm -rf /var/lib/apt/lists/*

# "We might need these for debugging someday"
```

**What this produces:**

```
Base image (ubuntu:22.04):    ~77 MB
After installing packages:    ~650 MB   ← 8.4× larger
Packages actually used:       python3, python3-pip
```

### Why It's Harmful

- Each unnecessary package is a potential CVE waiting to be reported
- `gcc`, `make`, `git` are commonly exploited by attackers after container compromise
- Image size increases significantly — longer pulls, more storage, slower deploys
- Vulnerability scanners report dozens of false-positive alerts from unused packages
- Recommended packages (`apt-get install -y`) pull in additional transitive dependencies

### Recommended Fix

```dockerfile
# Good: only production dependencies, no extras
FROM python:3.12-slim

# --no-install-recommends prevents transitive bloat
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      libpq5 \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r /app/requirements.txt

COPY . /app
WORKDIR /app

USER 1001
CMD ["python", "app.py"]
```

**For debugging, use ephemeral debug containers instead of baking tools into the image:**

```bash
# Kubernetes: attach a debug container to a running pod
kubectl debug -it my-pod --image=busybox:1.36 --target=my-container

# Docker: start a temporary container sharing the network namespace
docker run -it --rm --network container:my-app \
  nicolaka/netshoot \
  tcpdump -i eth0 port 8080
```

### Quick Check

- [ ] `apt-get install` uses `--no-install-recommends`
- [ ] No debugging tools (vim, curl, strace, htop) in production images
- [ ] No compilers or build tools (gcc, make, git) in the final stage
- [ ] `apt-get update` and `apt-get install` are in the same `RUN` layer
- [ ] Package lists are cleaned: `rm -rf /var/lib/apt/lists/*`

---

## 8. Not Setting Resource Limits

### Problem

Running containers without CPU and memory limits means a single misbehaving container can consume all host resources — starving every other container on the same host. Without limits, a memory leak goes undetected until the Linux OOM killer terminates random processes, and a CPU-intensive loop can make the entire host unresponsive.

### Example of the Bad Pattern

```yaml
# Bad: no resource limits at all
services:
  api:
    image: mycompany/api:1.4.2
    # No mem_limit, no cpus, no reservations
    # This container can consume ALL host resources

  worker:
    image: mycompany/worker:1.4.2
    # Worker processing large files — can eat 32 GB of RAM
```

```bash
# Running without limits — container can use all 64 GB of host RAM
docker run -d mycompany/api:1.4.2
```

**What happens when a container with no limits has a memory leak:**

```
┌──────────────────────────────────────────────────────────────────┐
│  Host: 16 GB RAM                                                 │
│                                                                  │
│  Time 0:  api (200 MB) + worker (500 MB) + db (2 GB) = 2.7 GB  │
│  Time 1h: api (1.2 GB)  — memory leak growing                   │
│  Time 3h: api (8.5 GB)  — still growing                         │
│  Time 5h: api (15 GB)   — host nearly exhausted                 │
│  Time 5h+: OOM killer fires → kills postgres (random victim)    │
│            → data corruption, cascading failure                  │
└──────────────────────────────────────────────────────────────────┘
```

### Why It's Harmful

- A single container can starve all other containers on the host
- OOM killer picks victims semi-randomly — your database may be killed instead of the leaking service
- CPU starvation causes latency spikes across unrelated services
- Without limits, capacity planning is impossible
- Memory leaks go undetected until the host runs out of memory
- Kubernetes scheduler cannot make informed placement decisions without resource requests

### Recommended Fix

```yaml
# Good: explicit resource limits and reservations
services:
  api:
    image: mycompany/api:1.4.2
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M

  worker:
    image: mycompany/worker:1.4.2
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 1G
        reservations:
          cpus: "0.5"
          memory: 512M
```

```bash
# CLI equivalent
docker run -d \
  --memory=512m \
  --memory-swap=512m \
  --cpus=1.0 \
  --pids-limit=256 \
  mycompany/api:1.4.2
```

**Kubernetes resource specification:**

```yaml
# k8s deployment — always set both requests and limits
containers:
  - name: api
    image: mycompany/api:1.4.2
    resources:
      requests:
        cpu: 250m
        memory: 256Mi
      limits:
        cpu: "1"
        memory: 512Mi
```

### Quick Check

- [ ] Every container in Compose has `deploy.resources.limits` for CPU and memory
- [ ] Every Kubernetes pod spec has `resources.requests` and `resources.limits`
- [ ] `--memory-swap` equals `--memory` (prevents swap usage)
- [ ] `--pids-limit` is set to prevent fork bombs
- [ ] Resource limits are validated under load — set based on actual profiling, not guesses

---

## 9. Ignoring Health Checks

### Problem

Without health checks, Docker (and orchestrators like Kubernetes) can only determine whether a container's main process is running — not whether the application inside is actually healthy and serving traffic. A container with a deadlocked thread pool, an exhausted database connection pool, or a crashed worker thread will appear "running" while silently failing to serve requests.

### Example of the Bad Pattern

```dockerfile
# Bad: no HEALTHCHECK instruction
FROM python:3.12-slim
COPY . /app
WORKDIR /app
RUN pip install --no-cache-dir -r requirements.txt
CMD ["gunicorn", "app:app", "--bind", "0.0.0.0:8000"]
# No HEALTHCHECK — Docker only knows if gunicorn PID is alive
```

```yaml
# Bad: no healthcheck in Compose
services:
  api:
    build: .
    ports:
      - "8000:8000"
    # No healthcheck — dependent services start immediately
    # regardless of whether the API is ready
```

**The failure scenario:**

```
1. Container starts — gunicorn process is running (Docker status: "running")
2. Application loads configuration — connects to database
3. Database connection pool is exhausted after 60 seconds
4. All requests return 503 — application is broken
5. Docker still reports container as "running" — no restart, no alert
6. Load balancer keeps sending traffic → users see 503 errors for hours
```

### Why It's Harmful

- Broken containers continue receiving traffic from load balancers
- Dependent services start before the dependency is actually ready
- Docker Compose `depends_on` without health checks only waits for the container to start, not for the application to be ready
- Auto-restart (`restart: always`) has nothing to trigger on — the process never exits
- Kubernetes readiness and liveness probes cannot function without application health endpoints

### Recommended Fix

```dockerfile
# Good: HEALTHCHECK in Dockerfile
FROM python:3.12-slim
COPY . /app
WORKDIR /app
RUN pip install --no-cache-dir -r requirements.txt

HEALTHCHECK --interval=15s --timeout=5s --start-period=30s --retries=3 \
  CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/healthz')"]

CMD ["gunicorn", "app:app", "--bind", "0.0.0.0:8000"]
```

```yaml
# Good: healthcheck in Compose with dependency conditions
services:
  api:
    build: .
    ports:
      - "8000:8000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/healthz"]
      interval: 15s
      timeout: 5s
      start_period: 30s
      retries: 3

  worker:
    build: ./worker
    depends_on:
      api:
        condition: service_healthy   # waits for API to be healthy
      redis:
        condition: service_healthy
```

**Implement a proper health endpoint in the application:**

```python
# app.py — health check endpoint
@app.route("/healthz")
def health():
    checks = {
        "database": check_database(),
        "redis": check_redis(),
        "disk_space": check_disk_space(),
    }
    status = "healthy" if all(checks.values()) else "unhealthy"
    code = 200 if status == "healthy" else 503
    return jsonify({"status": status, "checks": checks}), code
```

### Quick Check

- [ ] Every production Dockerfile has a `HEALTHCHECK` instruction
- [ ] Docker Compose services define `healthcheck` with appropriate intervals
- [ ] `depends_on` uses `condition: service_healthy` (not the default)
- [ ] Application exposes a `/healthz` or `/health` endpoint that checks dependencies
- [ ] `docker ps` shows container health status as "healthy" after startup

---

## 10. Modifying Running Containers

### Problem

Using `docker exec` to make changes inside running production containers — installing packages, editing configuration files, applying hotfixes, or patching code — creates "snowflake" containers that are impossible to reproduce. The next time the container restarts, all changes are lost, leading to mysterious "it worked before the restart" failures.

### Example of the Bad Pattern

```bash
# Bad: hotfixing production via docker exec
$ docker exec -it production-api bash

root@abc123:/app# apt-get update && apt-get install -y vim
root@abc123:/app# vim /app/config.py
# ... edit database connection pool size ...

root@abc123:/app# pip install new-dependency==2.1.0
root@abc123:/app# python -c "import new_dependency; print('works')"

root@abc123:/app# exit

# "Fixed! Ship it!"
# ... container restarts next Tuesday due to host maintenance ...
# ... all changes are gone ...
# ... frantic debugging ensues ...
```

**Infrastructure drift:**

```
┌────────────────────────────────┐
│ Container A (modified)         │  ← has vim, new dependency, config patch
│ Container B (original image)   │  ← missing all modifications
│ Container C (original image)   │  ← missing all modifications
└────────────────────────────────┘

Load balancer sends 33% of traffic to each container.
Only Container A works correctly — the rest return errors.
Nobody knows why 66% of requests fail.
```

### Why It's Harmful

- Changes are lost on container restart — the image is immutable; the container layer is ephemeral
- Infrastructure drift: containers from the same image behave differently
- No audit trail — `docker exec` changes are not logged or version-controlled
- Impossible to reproduce in staging or local development
- Violates Infrastructure as Code — the Dockerfile is no longer the source of truth
- Debugging requires `docker exec` into each container to compare states

### Recommended Fix

```bash
# Good: update the Dockerfile, build a new image, deploy
# 1. Make the change in the Dockerfile or application code
$ vim Dockerfile
$ vim app/config.py

# 2. Build a new image with a new tag
$ docker build -t mycompany/api:1.4.3 .

# 3. Run tests against the new image
$ docker run --rm mycompany/api:1.4.3 pytest

# 4. Push and deploy
$ docker push mycompany/api:1.4.3
$ docker compose up -d --no-deps api
```

**Treat containers as cattle, not pets:**

```
Pets (bad):                              Cattle (good):
┌─────────────────────┐                  ┌─────────────────────┐
│ "production-api-01" │                  │ api-7f8b9c (v1.4.3) │
│ Manually configured │                  │ Built from Dockerfile│
│ Hand-patched        │                  │ Immutable            │
│ Irreplaceable       │                  │ Disposable           │
│ Months of uptime    │                  │ Replaced on deploy   │
└─────────────────────┘                  └─────────────────────┘
```

### Quick Check

- [ ] `docker exec` is used only for debugging, never for production changes
- [ ] All configuration changes go through the Dockerfile or Compose file
- [ ] CI/CD pipeline is the only path to production deployment
- [ ] Containers are restarted periodically to verify they work from a clean image
- [ ] No manual package installations or file edits inside running containers

---

## 11. Hardcoding Configuration

### Problem

Embedding environment-specific values — hostnames, ports, feature flags, database connection strings, log levels — directly in the Dockerfile or application code means the same image cannot be used across development, staging, and production. A separate image must be built for each environment, violating the "build once, deploy everywhere" principle.

### Example of the Bad Pattern

```dockerfile
# Bad: environment-specific values baked into the image
FROM node:20-slim

WORKDIR /app
COPY . .
RUN npm ci --production

# Hardcoded for production — cannot reuse this image for staging
ENV NODE_ENV=production
ENV DATABASE_HOST=prod-db.internal.mycompany.com
ENV DATABASE_PORT=5432
ENV REDIS_HOST=prod-redis.internal.mycompany.com
ENV LOG_LEVEL=warn
ENV API_URL=https://api.mycompany.com
ENV FEATURE_NEW_CHECKOUT=true

CMD ["node", "server.js"]
```

**The result: one image per environment:**

```
mycompany/api:1.4.2-dev        ← DATABASE_HOST=dev-db.internal...
mycompany/api:1.4.2-staging    ← DATABASE_HOST=staging-db.internal...
mycompany/api:1.4.2-prod       ← DATABASE_HOST=prod-db.internal...
```

### Why It's Harmful

- The image tested in staging is not the same image deployed to production
- Configuration drift between environments causes "works in staging, broken in prod" bugs
- Changing a single configuration value requires rebuilding and redeploying the entire image
- Feature flags require a full build/deploy cycle to toggle
- Three images per version triples registry storage and CI build time
- Secrets and internal hostnames are baked into the image (see Anti-Pattern #3)

### Recommended Fix

```dockerfile
# Good: configuration-free image — all config injected at runtime
FROM node:20-slim

WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY . .

# Sensible defaults for local development only
# Production values are injected via environment variables at runtime
ENV NODE_ENV=production

USER 1001
EXPOSE 3000
CMD ["node", "server.js"]
```

```yaml
# docker-compose.yml — environment-specific config at deploy time
services:
  api:
    image: mycompany/api:1.4.2    # same image for all environments
    env_file:
      - .env.${DEPLOY_ENV:-dev}   # .env.dev, .env.staging, .env.prod
    environment:
      - LOG_LEVEL=${LOG_LEVEL:-info}
      - FEATURE_NEW_CHECKOUT=${FEATURE_NEW_CHECKOUT:-false}
```

```bash
# Application reads config from environment variables
# .env.prod
DATABASE_HOST=prod-db.internal.mycompany.com
DATABASE_PORT=5432
REDIS_HOST=prod-redis.internal.mycompany.com
LOG_LEVEL=warn
API_URL=https://api.mycompany.com
```

**The 12-Factor App configuration principle:**

```
Build:   docker build -t mycompany/api:1.4.2 .    ← one image
Ship:    docker push mycompany/api:1.4.2           ← one push
Run:
  Dev:     docker run -e DATABASE_HOST=localhost mycompany/api:1.4.2
  Staging: docker run -e DATABASE_HOST=staging-db mycompany/api:1.4.2
  Prod:    docker run -e DATABASE_HOST=prod-db    mycompany/api:1.4.2
```

### Quick Check

- [ ] The same image tag is deployed to all environments (dev, staging, production)
- [ ] No environment-specific hostnames, URLs, or ports in the Dockerfile
- [ ] Configuration is injected via environment variables or mounted config files
- [ ] Feature flags can be toggled without rebuilding the image
- [ ] `docker inspect` shows only sensible defaults, not production values

---

## 12. Not Cleaning Up

### Problem

Over time, Docker hosts accumulate stopped containers, dangling images (untagged intermediate layers), unused volumes, and orphaned networks. Without regular cleanup, disk space is consumed until the host runs out, causing builds to fail, containers to crash, and CI runners to stall.

### Example of the Bad Pattern

```bash
# A typical Docker host after 6 months without cleanup
$ docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          347       12        45.2GB    38.7GB (85%)
Containers      891       5         12.3GB    12.1GB (98%)
Local Volumes   234       8         67.8GB    58.2GB (85%)
Build Cache     -         -         23.4GB    23.4GB

# Total: 148.7 GB consumed — 132.4 GB reclaimable (89%)
```

**How it accumulates:**

```
Week 1:    Build 5 images, run 10 containers, create 3 volumes       →  2 GB
Week 4:    Old containers stopped but not removed                     →  8 GB
Month 3:   100+ dangling images from CI builds                        → 25 GB
Month 6:   Volumes from deleted containers still consuming space      → 68 GB
Month 6+:  "No space left on device" — builds fail, containers crash  → outage
```

### Why It's Harmful

- Disk space exhaustion causes container crashes and build failures
- CI/CD runners accumulate build cache and dangling images until they stall
- Unused volumes may contain sensitive data from old containers
- `docker pull` fails with "no space left on device" during production deployments
- Old images with known CVEs remain on the host
- Host performance degrades as the filesystem fills up

### Recommended Fix

```bash
# Manual cleanup — reclaim all unused resources
docker system prune -af --volumes

# Targeted cleanup
docker container prune -f           # remove stopped containers
docker image prune -af              # remove unused images (not just dangling)
docker volume prune -f              # remove unused volumes
docker network prune -f             # remove unused networks
docker builder prune -af            # remove build cache
```

**Automated cleanup with a cron job or systemd timer:**

```bash
# /etc/cron.daily/docker-cleanup
#!/bin/bash
# Remove containers stopped more than 24 hours ago
docker container prune -f --filter "until=24h"

# Remove images not used in the last 7 days
docker image prune -af --filter "until=168h"

# Remove dangling volumes (no container reference)
docker volume prune -f

# Remove build cache older than 7 days
docker builder prune -af --filter "until=168h"

# Log the current disk usage
docker system df >> /var/log/docker-cleanup.log
```

**Docker Compose cleanup:**

```bash
# Remove containers, networks, and volumes for a specific project
docker compose down --volumes --remove-orphans

# Remove images built by Compose as well
docker compose down --rmi local --volumes --remove-orphans
```

**CI/CD pipeline cleanup step:**

```yaml
# .github/workflows/build.yml — cleanup after build
jobs:
  build:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .
      - name: Push image
        run: docker push myapp:${{ github.sha }}
      - name: Cleanup
        if: always()
        run: |
          docker image prune -f --filter "until=24h"
          docker builder prune -f --filter "until=24h"
```

### Quick Check

- [ ] `docker system df` shows < 20% reclaimable space
- [ ] Automated cleanup runs daily on all Docker hosts (cron or systemd timer)
- [ ] CI/CD runners clean up dangling images and build cache after each build
- [ ] `docker compose down --volumes` is used when tearing down environments
- [ ] Monitoring alerts when Docker disk usage exceeds 80%

---

## Quick Reference Checklist

Use this checklist before deploying any containerised service to production.

### Security

- [ ] All Dockerfiles use a `USER` instruction to run as non-root
- [ ] No secrets in Dockerfiles (`ENV`, `COPY .env`, `ARG` with passwords)
- [ ] `docker inspect` shows no sensitive environment variables
- [ ] BuildKit `--mount=type=secret` used for build-time secrets
- [ ] Base images pinned to specific versions (no `:latest`)
- [ ] Images scanned with `trivy`, `docker scout`, or `dockle` in CI
- [ ] `--cap-drop ALL` used at runtime; only necessary capabilities added back
- [ ] `--read-only` filesystem where possible

### Image Build

- [ ] Multi-stage builds used for all compiled languages
- [ ] Final stage uses `alpine`, `slim`, `distroless`, or `scratch`
- [ ] No compilers, build tools, debugging utilities in production images
- [ ] `apt-get install` uses `--no-install-recommends`
- [ ] Package lists cleaned: `rm -rf /var/lib/apt/lists/*`
- [ ] `.dockerignore` excludes `.git/`, `node_modules/`, `.env`, test files
- [ ] Build context size < 50 MB
- [ ] Production images < 200 MB (language dependent)

### Runtime

- [ ] CPU and memory limits set for every container
- [ ] `--memory-swap` equals `--memory` to prevent swap
- [ ] `--pids-limit` set to prevent fork bombs
- [ ] One process per container — no supervisord
- [ ] Containers are immutable — no `docker exec` changes in production
- [ ] Health checks defined in Dockerfile or Compose
- [ ] `depends_on` uses `condition: service_healthy`

### Configuration and Operations

- [ ] Same image deployed to all environments (dev, staging, production)
- [ ] Configuration injected via environment variables or mounted files
- [ ] No hardcoded hostnames, ports, or feature flags in images
- [ ] Automated cleanup runs daily on Docker hosts
- [ ] CI runners clean up after each build
- [ ] `docker system df` monitored — alert at 80% disk usage

---

## Next Steps

1. **Score yourself** — Use the [Quick Reference Checklist](#quick-reference-checklist) and score your current Dockerfiles and Compose stacks. Any unchecked item is a potential anti-pattern.
2. **Fix critical severity first** — Address 🔴 Critical anti-patterns (#1, #2, #3) before high and medium ones.
3. **Read the companion guide** — [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) describes the correct patterns to replace these anti-patterns.
4. **Secure your images** — [07-SECURITY.md](07-SECURITY.md) covers container security in depth, including image scanning and runtime hardening.
5. **Optimise performance** — [08-PERFORMANCE.md](08-PERFORMANCE.md) covers resource limits, cgroup tuning, and build performance.
6. **Follow the learning path** — [LEARNING-PATH.md](LEARNING-PATH.md) provides a structured curriculum through all Docker topics.

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-01-01 | Initial document — 12 anti-patterns with examples and fixes | Platform Team |
| 1.1 | 2025-01-15 | Added quick reference checklist, image size comparison table | DevOps Team |
| 1.2 | 2025-02-01 | Expanded secrets section with BuildKit examples | Security Team |
| 1.3 | 2025-02-15 | Added CI/CD cleanup examples, Kubernetes resource specs | Platform Team |
