# Docker & Containers Learning Path

A structured, self-paced training guide to mastering Docker and containers — from foundations and container architecture to production-grade multi-service deployments, security hardening, and orchestration readiness. Each phase builds on the previous one, progressing from core concepts to advanced production patterns.

> **Time Estimate:** 8–10 weeks at ~5 hours/week. Adjust pace to your experience level. Engineers with prior Linux or virtualisation experience may complete Phase 1 in half the time.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds directly on prior knowledge; jumping ahead creates gaps that slow you down later
2. **Read the linked documents** — they contain the detailed content, code examples, and reference tables
3. **Complete the exercises** — hands-on practice is how container concepts become intuition; do not skip them
4. **Check yourself** — use the knowledge check items before moving to the next phase; if you cannot answer them confidently, re-read the relevant sections
5. **Build the capstone project** — the final project ties together all six phases and produces a portfolio artifact you can reference in engineering conversations

---

## Phase 1: Foundations (Week 1–2)

### Learning Objectives

- Understand the difference between containers and virtual machines
- Learn Docker architecture: Docker Engine, containerd, runc, and the OCI specification
- Grasp the container lifecycle: create, start, run, stop, remove
- Master essential Docker CLI commands for images and containers
- Understand what happens under the hood: namespaces, cgroups, and union filesystems

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | What are containers, Docker architecture, OCI standards, containers vs VMs |
| 2 | [02-CONTAINER-RUNTIME](02-CONTAINER-RUNTIME.md) | Container lifecycle, namespaces, cgroups, runc, containerd |

### Exercises

**1. Container vs VM Analysis:**

Choose a system you are familiar with — a personal project, a work service, or an open-source application — and compare how it would run as a container versus a VM:

- List what resources are shared with the host (kernel, filesystem, network) in each model
- For each of the following, state whether it is handled by the host kernel or the container:
  - Process isolation: Are processes visible to the host?
  - Filesystem: Does the container have its own root filesystem?
  - Networking: Does the container have its own IP address?
  - Resource limits: How is CPU and memory constrained?
- Estimate the startup time difference between a VM and a container for this application
- Identify three scenarios where a VM is still the better choice

Document your findings. You will revisit this analysis after Phase 6.

**2. Docker CLI Fundamentals:**

Run through the following Docker CLI exercises on your local machine:

```bash
# Pull and inspect an image
docker pull nginx:1.25-alpine
docker image inspect nginx:1.25-alpine --format='{{.Os}}/{{.Architecture}}'
docker image history nginx:1.25-alpine

# Run a container in the foreground and background
docker run --rm -it alpine:3.19 sh -c "echo 'Hello from Alpine' && cat /etc/os-release"
docker run -d --name web -p 8080:80 nginx:1.25-alpine

# Inspect the running container
docker ps
docker inspect web --format='{{.State.Status}}'
docker logs web
docker top web
docker stats web --no-stream

# Execute commands inside the running container
docker exec web nginx -v
docker exec -it web sh -c "ls -la /usr/share/nginx/html"

# Stop, restart, and remove
docker stop web
docker start web
docker rm -f web
```

After completing each command, answer:
- What is the difference between `docker run` and `docker create` + `docker start`?
- What does `--rm` do, and when should you use it?
- What is the difference between `docker stop` (SIGTERM) and `docker kill` (SIGKILL)?
- How does `-p 8080:80` map ports, and what happens if port 8080 is already in use?

**3. Exploring Container Internals:**

Run a container and explore the Linux primitives that power it:

```bash
# Run a long-lived container
docker run -d --name explorer alpine:3.19 sleep 3600

# Find the container's PID on the host
docker inspect explorer --format='{{.State.Pid}}'

# View the namespaces (requires root on Linux host)
# Replace <PID> with the output from the previous command
sudo ls -la /proc/<PID>/ns/

# View the cgroup limits
cat /sys/fs/cgroup/docker/$(docker inspect explorer --format='{{.Id}}')/memory.max 2>/dev/null || \
  echo "Check cgroup v2 path on your system"

# View the overlay filesystem
docker inspect explorer --format='{{.GraphDriver.Data.MergedDir}}'
```

### Knowledge Check

- [ ] What are the three Linux kernel features that enable containers (namespaces, cgroups, union filesystems)?
- [ ] How is a container different from a virtual machine in terms of kernel sharing?
- [ ] What is the OCI specification, and what two components does it define (runtime-spec, image-spec)?
- [ ] What does `docker run` do under the hood (pull image if needed, create container, start container)?

---

## Phase 2: Images & Building (Week 3–4)

### Learning Objectives

- Understand Docker image structure: layers, manifests, and content-addressable storage
- Write production-quality Dockerfiles with proper layer ordering and caching
- Implement multi-stage builds to minimise final image size
- Optimise build performance with layer caching, `.dockerignore`, and BuildKit features
- Understand image tagging strategies and when to use digests

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [01-IMAGES-AND-DOCKERFILES](01-IMAGES-AND-DOCKERFILES.md) | Image layers, Dockerfile instructions, build context, caching |
| 2 | [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | Anti-patterns #4 (Fat Images), #6 (No .dockerignore), #7 (Unnecessary Packages) |

### Exercises

**1. Layer Optimisation Challenge:**

Start with the following intentionally poorly ordered Dockerfile and optimise it for build caching:

```dockerfile
# Bad: every code change invalidates the dependency install layer
FROM python:3.12-slim
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
RUN apt-get update && apt-get install -y libpq5
CMD ["python", "app.py"]
```

Rewrite the Dockerfile to:
- Install OS dependencies before copying application code
- Copy and install Python dependencies before copying the rest of the code
- Use `--no-cache-dir` with pip
- Clean up apt lists
- Add a `.dockerignore` file

After rewriting, test the cache behaviour:
1. Build the image — note the build time
2. Change a line in `app.py` — rebuild and note that dependency layers are cached
3. Add a new dependency to `requirements.txt` — rebuild and note which layers are invalidated

**2. Multi-Stage Build for a Go Application:**

Write a multi-stage Dockerfile for a Go web server that results in an image under 15 MB:

```go
// main.go
package main

import (
    "fmt"
    "net/http"
    "os"
)

func main() {
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello from Go container!")
    })
    http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        fmt.Fprintf(w, `{"status": "healthy"}`)
    })
    fmt.Printf("Server starting on port %s\n", port)
    http.ListenAndServe(":"+port, nil)
}
```

Your Dockerfile should:
- Use `golang:1.22-alpine` as the builder stage
- Use `gcr.io/distroless/static-debian12:nonroot` as the final stage
- Compile with `CGO_ENABLED=0` and `-ldflags="-s -w"` to strip debug symbols
- Run as a non-root user
- Include a `HEALTHCHECK` instruction

After building, compare the image sizes:
```bash
# Compare single-stage vs multi-stage
docker images | grep myapp
```

**3. Dockerfile Linting and Scanning:**

Install and run Dockerfile linting and image scanning tools:

```bash
# Lint the Dockerfile with hadolint
docker run --rm -i hadolint/hadolint < Dockerfile

# Scan the built image for vulnerabilities
docker scout cves myapp:latest
# Or use trivy
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image myapp:latest
```

Fix any warnings or vulnerabilities identified by the tools. Document:
- How many lint warnings were found and what they recommended
- How many CVEs were found in the original image vs the optimised multi-stage image
- Which base image choice resulted in the fewest vulnerabilities

### Knowledge Check

- [ ] What is a Docker image layer, and why does the order of Dockerfile instructions matter for caching?
- [ ] What is the difference between `COPY` and `ADD` in a Dockerfile?
- [ ] How does a multi-stage build reduce final image size?
- [ ] Why should you pin base image versions instead of using `:latest`?

---

## Phase 3: Networking & Storage (Week 5–6)

### Learning Objectives

- Understand Docker networking models: bridge, host, none, overlay, and macvlan
- Configure container-to-container communication using custom bridge networks
- Implement DNS-based service discovery in Docker networks
- Master Docker storage: volumes, bind mounts, and tmpfs mounts
- Understand the differences between volumes and bind mounts and when to use each

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [03-NETWORKING](03-NETWORKING.md) | Bridge networks, overlay networks, DNS resolution, port mapping, network drivers |
| 2 | [04-STORAGE-AND-VOLUMES](04-STORAGE-AND-VOLUMES.md) | Volumes, bind mounts, tmpfs, volume drivers, data persistence |

### Exercises

**1. Custom Bridge Network and Service Discovery:**

Create a custom bridge network and explore DNS-based container communication:

```bash
# Create a custom bridge network
docker network create --driver bridge my-app-net

# Run two containers on the custom network
docker run -d --name api --network my-app-net nginx:1.25-alpine
docker run -d --name db --network my-app-net postgres:16-alpine \
  -e POSTGRES_PASSWORD=devpassword

# Test DNS resolution — containers can reach each other by name
docker exec api ping -c 3 db
docker exec api nslookup db

# Inspect the network
docker network inspect my-app-net --format='{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{"\n"}}{{end}}'
```

After completing, answer:
- What is the difference between the default `bridge` network and a user-defined bridge network?
- Why can containers on the default bridge network not resolve each other by name?
- What IP range does Docker assign to the custom bridge network?

**2. Volumes and Data Persistence:**

Explore the three Docker storage types and verify data persistence across container restarts:

```bash
# Named volume — managed by Docker, persists after container removal
docker volume create pgdata
docker run -d --name db1 \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=devpassword \
  postgres:16-alpine

# Write data, then remove the container
docker exec db1 psql -U postgres -c "CREATE TABLE test (id serial, name text);"
docker exec db1 psql -U postgres -c "INSERT INTO test (name) VALUES ('persistent');"
docker rm -f db1

# Start a new container with the same volume — data survives
docker run -d --name db2 \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=devpassword \
  postgres:16-alpine

docker exec db2 psql -U postgres -c "SELECT * FROM test;"
# Output: persistent — data survived container removal

# Bind mount — maps a host directory into the container
docker run -d --name dev-api \
  -v $(pwd)/src:/app/src:ro \
  -p 3000:3000 \
  node:20-alpine

# tmpfs mount — in-memory, never written to disk
docker run -d --name secure-app \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  myapp:1.0.0
```

After completing, answer:
- Where does Docker store named volume data on the host filesystem?
- What happens to bind mount data when the container is removed?
- When should you use tmpfs instead of a volume?

**3. Multi-Container Network Architecture:**

Build the following network architecture using only Docker CLI commands (no Compose):

```
┌──────────────────────────────────────────────────┐
│                 frontend-net                      │
│                                                   │
│  ┌──────────┐              ┌──────────────────┐  │
│  │  nginx    │──────────── │  api              │  │
│  │  :80      │              │  :8000            │  │
│  └──────────┘              └────────┬─────────┘  │
│                                      │            │
└──────────────────────────────────────┼────────────┘
                                       │
┌──────────────────────────────────────┼────────────┐
│                 backend-net          │            │
│                                      │            │
│                             ┌────────▼─────────┐  │
│  ┌──────────┐              │  api              │  │
│  │ postgres  │◄─────────── │  (also on this    │  │
│  │ :5432     │              │   network)        │  │
│  └──────────┘              └──────────────────┘  │
│                                                   │
│  ┌──────────┐                                    │
│  │  redis    │                                    │
│  │  :6379    │                                    │
│  └──────────┘                                    │
│                                                   │
└──────────────────────────────────────────────────┘
```

Requirements:
- `nginx` can reach `api` but not `postgres` or `redis`
- `api` can reach both `nginx` (frontend-net) and `postgres`/`redis` (backend-net)
- `postgres` and `redis` cannot reach `nginx`
- Only `nginx` port 80 is published to the host

### Knowledge Check

- [ ] What is the difference between a bridge network and an overlay network?
- [ ] How does Docker DNS resolution work in user-defined bridge networks?
- [ ] What is the difference between a named volume and a bind mount?
- [ ] When should you use `--tmpfs` instead of a volume?

---

## Phase 4: Compose & Multi-Container (Week 7)

### Learning Objectives

- Write Docker Compose files to define multi-container applications
- Understand service orchestration: dependency ordering, health checks, restart policies
- Configure environment variables, secrets, and volumes in Compose
- Master Compose commands: `up`, `down`, `build`, `logs`, `exec`, `scale`
- Understand container registries: pushing, pulling, tagging, and image management

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [05-DOCKER-COMPOSE](05-DOCKER-COMPOSE.md) | Compose file syntax, services, networks, volumes, profiles, overrides |
| 2 | [06-REGISTRIES](06-REGISTRIES.md) | Docker Hub, GitHub Container Registry, private registries, image tagging |

### Exercises

**1. Build a Complete Application Stack with Compose:**

Write a `docker-compose.yml` that runs a three-tier web application:

```yaml
# docker-compose.yml — complete application stack
services:
  frontend:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      api:
        condition: service_healthy
    networks:
      - frontend-net

  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    expose:
      - "8000"
    environment:
      - DATABASE_URL=postgres://app:${DB_PASSWORD}@db:5432/myapp
      - REDIS_URL=redis://redis:6379
      - LOG_LEVEL=${LOG_LEVEL:-info}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/healthz"]
      interval: 10s
      timeout: 5s
      start_period: 30s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - frontend-net
      - backend-net

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=app
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend-net

  redis:
    image: redis:7.2-alpine
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
    networks:
      - backend-net

volumes:
  pgdata:
  redis-data:

networks:
  frontend-net:
  backend-net:
```

After writing the Compose file:
1. Start the stack: `docker compose up -d`
2. Verify all services are healthy: `docker compose ps`
3. View logs for a specific service: `docker compose logs -f api`
4. Scale the API to 3 replicas: `docker compose up -d --scale api=3`
5. Tear down everything including volumes: `docker compose down -v`

**2. Compose Profiles and Environment Overrides:**

Create a Compose setup that supports multiple environments using profiles and overrides:

```yaml
# docker-compose.yml — base configuration
services:
  api:
    build: ./api
    environment:
      - LOG_LEVEL=info

# docker-compose.override.yml — local development additions (auto-loaded)
services:
  api:
    volumes:
      - ./api/src:/app/src   # hot-reload source code
    environment:
      - LOG_LEVEL=debug
    ports:
      - "8000:8000"          # expose for local testing
      - "5678:5678"          # debugger port

# docker-compose.prod.yml — production overrides
services:
  api:
    image: mycompany/api:${IMAGE_TAG}
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 512M
    restart: unless-stopped
```

Test the environment configurations:
```bash
# Local development (uses override automatically)
docker compose up -d

# Production (explicit file list)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

**3. Push to a Container Registry:**

Tag and push your application image to a container registry:

```bash
# Build and tag
docker build -t myapp:1.0.0 ./api

# Tag for registry
docker tag myapp:1.0.0 ghcr.io/myorg/myapp:1.0.0
docker tag myapp:1.0.0 ghcr.io/myorg/myapp:latest

# Login and push
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin
docker push ghcr.io/myorg/myapp:1.0.0
docker push ghcr.io/myorg/myapp:latest

# Verify the image is available
docker pull ghcr.io/myorg/myapp:1.0.0
```

### Knowledge Check

- [ ] What is the difference between `depends_on` with and without `condition: service_healthy`?
- [ ] How does `docker-compose.override.yml` work, and when is it loaded automatically?
- [ ] What is the difference between `ports` and `expose` in a Compose file?
- [ ] What is a container registry, and why should you use a private registry for production images?

---

## Phase 5: Security & Performance (Week 8)

### Learning Objectives

- Understand container security: image scanning, runtime hardening, and least-privilege principles
- Implement non-root containers, read-only filesystems, and capability dropping
- Configure resource limits (CPU, memory, PIDs) and understand cgroup behaviour
- Optimise build performance with BuildKit, caching strategies, and parallel builds
- Scan images for vulnerabilities and enforce security policies in CI/CD

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [07-SECURITY](07-SECURITY.md) | Image scanning, non-root users, secrets management, runtime security, CIS benchmarks |
| 2 | [08-PERFORMANCE](08-PERFORMANCE.md) | Resource limits, cgroups, build cache, BuildKit, monitoring container performance |

### Exercises

**1. Security Hardening Audit:**

Take the following insecure Dockerfile and fix every security issue:

```dockerfile
# Insecure: fix all issues
FROM ubuntu:latest
RUN apt-get update && apt-get install -y curl vim gcc python3 python3-pip
ENV DB_PASSWORD=supersecret123
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
EXPOSE 8000
CMD ["python3", "app.py"]
```

Your hardened Dockerfile should address:
- Pin the base image to a specific version and use a minimal variant
- Remove unnecessary packages (vim, gcc, curl)
- Remove the hardcoded secret
- Use `--no-install-recommends` and clean up apt lists
- Add a non-root user
- Add a `HEALTHCHECK`
- Use multi-stage build if compilation is needed

After hardening, scan both images:
```bash
# Scan the original
docker build -t insecure -f Dockerfile.insecure .
docker scout cves insecure

# Scan the hardened version
docker build -t hardened -f Dockerfile.hardened .
docker scout cves hardened

# Compare CVE counts
```

**2. Resource Limits and Cgroup Behaviour:**

Explore how Docker enforces resource limits using cgroups:

```bash
# Run a container with memory limit
docker run -d --name limited \
  --memory=128m \
  --memory-swap=128m \
  --cpus=0.5 \
  --pids-limit=64 \
  python:3.12-slim sleep 3600

# Inspect the cgroup limits
docker inspect limited --format='{{.HostConfig.Memory}}'
docker inspect limited --format='{{.HostConfig.NanoCpus}}'
docker stats limited --no-stream

# Test the memory limit — this container will be OOM-killed
docker run --rm --memory=64m python:3.12-slim python3 -c "
data = []
while True:
    data.append('x' * 10_000_000)  # allocate 10 MB chunks
"
# Expected: container exits with OOM kill (exit code 137)
```

After the exercise, answer:
- What exit code indicates an OOM kill?
- What is the difference between `--memory` and `--memory-swap`?
- What does `--cpus=0.5` mean in terms of CPU scheduling?

**3. Build Performance Optimisation:**

Measure and optimise Docker build performance using BuildKit:

```bash
# Enable BuildKit (default in Docker 23+)
export DOCKER_BUILDKIT=1

# Build with timing output
docker build --progress=plain -t myapp:1.0.0 . 2>&1 | tee build.log

# Analyse cache usage
docker buildx du

# Use cache mounts for package managers
# Dockerfile with cache mounts:
```

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim

RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt \
    apt-get update && apt-get install -y --no-install-recommends libpq5

RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

### Knowledge Check

- [ ] Why should containers run as non-root, and how do you verify the running user?
- [ ] What is the difference between `--cap-drop ALL` and `--privileged`?
- [ ] What exit code does a container return when it is killed by the OOM killer?
- [ ] How do BuildKit cache mounts (`--mount=type=cache`) improve build speed?

---

## Phase 6: Production & Best Practices (Week 9–10)

### Learning Objectives

- Apply Docker best practices across images, builds, runtime, and operations
- Identify and remediate all 12 anti-patterns from the anti-patterns document
- Build a production-readiness checklist for containerised services
- Understand when and how to transition from Docker Compose to Kubernetes
- Implement logging, monitoring, and observability for containers in production

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Full document — Dockerfile standards, image hygiene, runtime configuration |
| 2 | [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | Full document — all 12 anti-patterns, quick reference checklist |

### Exercises

**1. Production Readiness Audit:**

Select a service (your own, the capstone, or an open-source project) and perform a full Docker audit against the anti-patterns checklist from [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md):

```
Security:
  [ ] Non-root user in Dockerfile (USER instruction)
  [ ] No secrets in image (ENV, COPY .env, ARG)
  [ ] Base images pinned to specific versions
  [ ] Images scanned in CI (trivy, docker scout, dockle)
  [ ] --cap-drop ALL at runtime

Image Build:
  [ ] Multi-stage build used
  [ ] Final stage uses alpine/slim/distroless/scratch
  [ ] No compilers or debugging tools in final image
  [ ] .dockerignore present and comprehensive
  [ ] Production image < 200 MB

Runtime:
  [ ] CPU and memory limits set
  [ ] One process per container
  [ ] Health checks defined
  [ ] Containers are immutable (no docker exec changes)
  [ ] depends_on uses condition: service_healthy

Operations:
  [ ] Same image for all environments
  [ ] Configuration via environment variables
  [ ] Automated cleanup on Docker hosts
  [ ] Logging to stdout/stderr (not files inside container)
```

For each unchecked item, write a one-paragraph remediation plan: what to change, estimated effort, and priority.

**2. Anti-Pattern Identification Exercise:**

Review the following system description and identify at least 5 anti-patterns from [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md):

> "We run a web application with a single Docker container that runs nginx, a Python API, a Celery worker, and Redis using supervisord. Our Dockerfile uses `FROM python:latest` and installs vim, curl, htop, and gcc for debugging. The database password is set with `ENV DB_PASS=prod_password_123` in the Dockerfile. We don't have a `.dockerignore` file — builds take 5 minutes because the 800 MB `.git` directory is sent as context. Our containers run as root. We frequently SSH into production containers to apply hotfixes. Images are tagged `:latest` and deployed to all environments. There are no health checks — we check if the container is running with `docker ps`. No resource limits are set. We've never run `docker system prune` and our CI runner has 200 GB of dangling images."

For each anti-pattern identified:
- State the anti-pattern name and number
- Quote the specific evidence from the description
- Write the recommended fix in 2–3 sentences

<details>
<summary>Anti-patterns identified (click to reveal suggested answers)</summary>

1. **#5 One Container, Many Processes** — "single Docker container that runs nginx, a Python API, a Celery worker, and Redis using supervisord"
2. **#2 Using :latest Tag in Production** — "FROM python:latest" and "Images are tagged :latest"
3. **#7 Installing Unnecessary Packages** — "installs vim, curl, htop, and gcc for debugging"
4. **#3 Storing Secrets in Images** — "ENV DB_PASS=prod_password_123 in the Dockerfile"
5. **#6 Not Using .dockerignore** — "don't have a .dockerignore file — builds take 5 minutes because the 800 MB .git directory"
6. **#1 Running as Root** — "Our containers run as root"
7. **#10 Modifying Running Containers** — "frequently SSH into production containers to apply hotfixes"
8. **#9 Ignoring Health Checks** — "no health checks — we check if the container is running with docker ps"
9. **#8 Not Setting Resource Limits** — "No resource limits are set"
10. **#12 Not Cleaning Up** — "never run docker system prune" and "200 GB of dangling images"
11. **#4 Fat Images** — implied by using `python:latest` (full image, not slim or multi-stage)
12. **#11 Hardcoding Configuration** — implied by ENV DB_PASS in Dockerfile (production values baked in)

</details>

**3. Orchestration Readiness Assessment:**

Evaluate whether your containerised application is ready to migrate from Docker Compose to Kubernetes:

```
Readiness Checklist:
  [ ] Images are stored in a container registry (not built locally)
  [ ] Health check endpoints implemented (/healthz, /readyz)
  [ ] Configuration via environment variables (not baked into images)
  [ ] Secrets managed externally (not in images or Compose files)
  [ ] Stateless application design (or volumes mapped to persistent claims)
  [ ] Resource limits defined and tested under load
  [ ] Logging to stdout/stderr (collected by container runtime)
  [ ] Graceful shutdown handles SIGTERM (not just SIGKILL)
  [ ] One process per container
  [ ] No hard-coded container names or IPs (use DNS service discovery)
```

### Knowledge Check

- [ ] What are the four categories from the anti-patterns checklist (Security, Image Build, Runtime, Operations) and name two rules from each?
- [ ] What is the "build once, deploy everywhere" principle, and which anti-patterns violate it?
- [ ] What is the difference between logging to a file inside a container and logging to stdout/stderr?
- [ ] What signals does Docker send when stopping a container, and how should your application handle them?

---

## Capstone Project: Containerise a Multi-Service Application

Design and implement a fully containerised multi-service application with production-grade Docker practices. Build a simple e-commerce system with the following services:

### System Architecture

```
                    ┌─────────────────────────┐
                    │     Load Balancer        │
                    │     (nginx)              │
                    │     :80                  │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │     API Gateway           │
                    │     (Node.js/Express)     │
                    │     :3000                 │
                    └──────┬──────────┬────────┘
                           │          │
              ┌────────────▼┐    ┌────▼──────────┐
              │  Product     │    │  Order         │
              │  Service     │    │  Service       │
              │  (Python)    │    │  (Go)          │
              │  :8001       │    │  :8002         │
              └──────┬───────┘    └──────┬────────┘
                     │                    │
              ┌──────▼───────┐    ┌──────▼────────┐
              │  PostgreSQL   │    │  PostgreSQL    │
              │  (products)   │    │  (orders)      │
              │  :5432        │    │  :5433         │
              └──────────────┘    └───────────────┘
                                          │
                                   ┌──────▼────────┐
                                   │    Redis       │
                                   │    (cache)     │
                                   │    :6379       │
                                   └───────────────┘
```

### Requirements

| Requirement | What to Implement |
|-------------|-------------------|
| **Dockerfiles** | Multi-stage build for Go and Node.js services; optimised Python Dockerfile; all images < 200 MB |
| **Security** | Non-root users in all Dockerfiles; no secrets in images; BuildKit secrets for private dependencies |
| **Compose** | Full `docker-compose.yml` with health checks, resource limits, named volumes, and custom networks |
| **Networking** | Separate frontend and backend networks; database containers not accessible from frontend |
| **Storage** | Named volumes for PostgreSQL data; bind mounts for local development hot-reload |
| **Configuration** | Environment variables for all config; `.env` file for local development; same images for all envs |
| **Health checks** | Every service implements `/healthz`; Compose uses `condition: service_healthy` |
| **Registry** | Push all images to a container registry with SemVer tags |
| **Anti-patterns** | Run the quick reference checklist from [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md); zero anti-patterns in final submission |
| **Documentation** | README with architecture diagram, setup instructions, and environment variable reference |

### Evaluation Criteria

| Area | What to Verify |
|------|----------------|
| **Dockerfiles** | Multi-stage builds used; images under 200 MB; hadolint passes with no warnings |
| **Security** | Non-root users; no secrets; `docker scout` reports minimal CVEs |
| **Compose** | `docker compose up` starts all services; all health checks pass within 60 seconds |
| **Networking** | Database containers not reachable from the nginx container (test with `docker exec`) |
| **Storage** | Data persists across `docker compose down && docker compose up` (without `-v`) |
| **Anti-patterns** | All 12 anti-patterns from the checklist evaluated; findings documented |
| **Documentation** | README explains how to run locally, environment variables, and architecture |

---

## Suggested Resources

### Books

| Book | Author | Focus |
|------|--------|-------|
| *Docker Deep Dive* | Nigel Poulton | Comprehensive Docker guide from fundamentals to production |
| *Docker in Action* | Jeff Nickoloff, Stephen Kuenzli | Practical Docker patterns and real-world scenarios |
| *Container Security* | Liz Rice | Linux primitives, image security, runtime hardening |
| *Building Secure and Reliable Systems* | Google SRE Team | Production infrastructure, security, and reliability |
| *The Docker Book* | James Turnbull | Getting started with Docker, CI/CD integration |
| *Kubernetes in Action* | Marko Lukša | Next step after Docker — container orchestration at scale |

### Online Resources

- **docs.docker.com** — Official Docker documentation, getting-started guides, Dockerfile reference
- **opencontainers.org** — OCI runtime and image specifications
- **github.com/docker/awesome-compose** — Official Docker Compose example applications
- **container.training** — Free, self-paced container training by Jérôme Petazzoni
- **cis-docker-benchmark** — CIS Docker Benchmark for security hardening
- **hadolint.github.io/hadolint** — Dockerfile linter with best practice rules

### Tools

| Tool | Purpose | Phase |
|------|---------|-------|
| Docker Desktop / Docker Engine | Container runtime | 1–6 |
| Docker Compose | Multi-container orchestration | 4–6, Capstone |
| BuildKit | Advanced image building, cache mounts, secrets | 2, 5 |
| hadolint | Dockerfile linting | 2, 6 |
| trivy / docker scout | Image vulnerability scanning | 2, 5, 6 |
| dockle | CIS benchmark scanning for images | 5, 6 |
| dive | Exploring image layers and wasted space | 2, 5 |
| ctop | Container resource monitoring (like htop) | 5 |
| lazydocker | Terminal UI for Docker management | 1–6 |
| kind / minikube | Local Kubernetes for orchestration readiness | 6, Capstone |

---

## Quick Reference: Document Map

| # | Document | Phase | Key Topics |
|---|----------|-------|------------|
| 00 | [OVERVIEW](00-OVERVIEW.md) | 1 | Container fundamentals, Docker architecture, OCI standards |
| 01 | [IMAGES-AND-DOCKERFILES](01-IMAGES-AND-DOCKERFILES.md) | 2 | Image layers, Dockerfile instructions, multi-stage builds |
| 02 | [CONTAINER-RUNTIME](02-CONTAINER-RUNTIME.md) | 1 | Container lifecycle, namespaces, cgroups, runc |
| 03 | [NETWORKING](03-NETWORKING.md) | 3 | Bridge, overlay, DNS discovery, port mapping |
| 04 | [STORAGE-AND-VOLUMES](04-STORAGE-AND-VOLUMES.md) | 3 | Volumes, bind mounts, tmpfs, data persistence |
| 05 | [DOCKER-COMPOSE](05-DOCKER-COMPOSE.md) | 4 | Compose syntax, services, profiles, overrides |
| 06 | [REGISTRIES](06-REGISTRIES.md) | 4 | Docker Hub, GHCR, private registries, tagging |
| 07 | [SECURITY](07-SECURITY.md) | 5 | Image scanning, non-root, secrets, CIS benchmarks |
| 08 | [PERFORMANCE](08-PERFORMANCE.md) | 5 | Resource limits, cgroups, BuildKit, monitoring |
| 09 | [BEST-PRACTICES](09-BEST-PRACTICES.md) | 6 | Dockerfile standards, image hygiene, production readiness |
| 10 | [ANTI-PATTERNS](10-ANTI-PATTERNS.md) | 6 | 12 anti-patterns, quick reference checklist |
| — | [LEARNING-PATH](LEARNING-PATH.md) | All | This document — structured 6-phase curriculum |

---

## Next Steps

1. **Complete the capstone** — The capstone project is the most important part of this learning path; it ties together every phase into a single deliverable.
2. **Audit your existing services** — Apply the [anti-patterns checklist](10-ANTI-PATTERNS.md#quick-reference-checklist) to your production Dockerfiles and Compose stacks.
3. **Learn Kubernetes** — Once you have mastered Docker, the natural next step is container orchestration with Kubernetes. See the [Kubernetes learning materials](../kubernetes/) in this repository.
4. **Explore observability** — Containerised services need monitoring, logging, and tracing. See the [Observability learning materials](../observability/) in this repository.
5. **Contribute** — Found an error, have a better example, or want to add a new exercise? Open a PR following the [CONTRIBUTING.md](../CONTRIBUTING.md) guide.

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-01-01 | Initial document — 6-phase learning path with exercises and capstone | Platform Team |
| 1.1 | 2025-01-15 | Added document map, suggested resources, evaluation criteria for capstone | DevOps Team |
| 1.2 | 2025-02-01 | Expanded Phase 5 with BuildKit cache mounts and security scanning exercises | Security Team |
| 1.3 | 2025-02-15 | Added orchestration readiness assessment, Phase 6 anti-pattern exercise | Platform Team |
