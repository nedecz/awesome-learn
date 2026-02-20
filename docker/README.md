# Docker & Containers Learning Resources

A comprehensive guide to Docker and containers — from foundational concepts and image building to networking, storage, security, and production-ready container patterns.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | What are containers, Docker architecture, OCI standards | **Start here** |
| [01-IMAGES-AND-DOCKERFILES](01-IMAGES-AND-DOCKERFILES.md) | Building images, Dockerfile best practices, multi-stage builds | When building container images |
| [02-CONTAINER-RUNTIME](02-CONTAINER-RUNTIME.md) | Docker Engine, containerd, runc, container lifecycle | When understanding runtime internals |
| [03-NETWORKING](03-NETWORKING.md) | Bridge, host, overlay networks, DNS, port mapping | When configuring container networking |
| [04-STORAGE-AND-VOLUMES](04-STORAGE-AND-VOLUMES.md) | Volumes, bind mounts, tmpfs, storage drivers | When managing persistent data |
| [05-DOCKER-COMPOSE](05-DOCKER-COMPOSE.md) | Multi-container apps, compose files, service dependencies | When orchestrating local environments |
| [06-REGISTRIES](06-REGISTRIES.md) | Docker Hub, ECR, ACR, GCR, private registries | When distributing container images |
| [07-SECURITY](07-SECURITY.md) | Image scanning, rootless containers, secrets management | **Essential — security hardening** |
| [08-PERFORMANCE](08-PERFORMANCE.md) | Resource limits, cgroups, image layer optimization | When tuning container performance |
| [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Production patterns, 12-factor apps, health checks | **Essential — production checklist** |
| [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | Common container mistakes and how to avoid them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured learning guide with exercises | **Start here** after the Overview |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand what containers are and why they matter
   - Learn Docker architecture (daemon, client, registry)
   - Explore OCI standards and the container ecosystem

2. **Build Your First Image** ([01-IMAGES-AND-DOCKERFILES](01-IMAGES-AND-DOCKERFILES.md))
   - Write a Dockerfile for a simple application
   - Understand image layers and caching
   - Use multi-stage builds to reduce image size

3. **Run and Connect Containers** ([03-NETWORKING](03-NETWORKING.md))
   - Map ports and connect containers on bridge networks
   - Understand Docker DNS and service discovery
   - Expose services to the host machine

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured curriculum with hands-on exercises
   - Progressive skill building from basics to production

### For Experienced Users

1. **Review Best Practices** ([09-BEST-PRACTICES](09-BEST-PRACTICES.md))
   - Production-ready container patterns
   - 12-factor app methodology with containers
   - Health check and graceful shutdown patterns

2. **Avoid Anti-Patterns** ([10-ANTI-PATTERNS](10-ANTI-PATTERNS.md))
   - Common container mistakes and misconfigurations
   - Image bloat, security oversights, and runtime pitfalls

3. **Harden Security** ([07-SECURITY](07-SECURITY.md))
   - Image scanning and vulnerability management
   - Rootless containers and least-privilege principles
   - Secrets management and supply chain security

4. **Optimize Performance** ([08-PERFORMANCE](08-PERFORMANCE.md))
   - Resource limits with cgroups v2
   - Image layer optimization and build caching
   - Container resource monitoring and profiling

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Docker Client                             │
│   docker build │ docker run │ docker push │ docker compose       │
└───────────────────────────┬─────────────────────────────────────┘
                            │ REST API (unix socket / TCP)
┌───────────────────────────▼─────────────────────────────────────┐
│                       Docker Daemon (dockerd)                    │
│   ┌─────────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│   │  Image Manager  │  │  Network     │  │  Volume Manager   │  │
│   │  (pull, build,  │  │  Manager     │  │  (volumes, bind   │  │
│   │   push, cache)  │  │  (bridge,    │  │   mounts, tmpfs)  │  │
│   └────────┬────────┘  │   overlay,   │  └───────────────────┘  │
│            │           │   host)      │                          │
│   ┌────────▼────────┐  └──────────────┘                          │
│   │   containerd    │                                            │
│   │  (container     │                                            │
│   │   lifecycle)    │                                            │
│   └────────┬────────┘                                            │
│            │                                                     │
│   ┌────────▼────────┐                                            │
│   │      runc       │                                            │
│   │  (OCI runtime)  │                                            │
│   └────────┬────────┘                                            │
└────────────│────────────────────────────────────────────────────┘
             │
┌────────────▼────────────────────────────────────────────────────┐
│                     Running Containers                           │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│   │ Container A  │  │ Container B  │  │ Container C  │          │
│   │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │          │
│   │ │  App     │ │  │ │  App     │ │  │ │  App     │ │          │
│   │ ├──────────┤ │  │ ├──────────┤ │  │ ├──────────┤ │          │
│   │ │  Image   │ │  │ │  Image   │ │  │ │  Image   │ │          │
│   │ │  Layers  │ │  │ │  Layers  │ │  │ │  Layers  │ │          │
│   │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │          │
│   └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
             │
┌────────────▼────────────────────────────────────────────────────┐
│                     Container Registry                           │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│   │  Docker Hub  │  │  ECR / ACR   │  │   Private    │          │
│   │  (public)    │  │  GCR (cloud) │  │   Registry   │          │
│   └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

## 🔑 Key Concepts

```
Container Fundamentals
──────────────────────
Image      → Read-only template with application code, runtime, and dependencies
Container  → Running instance of an image with its own writable layer
Dockerfile → Declarative recipe for building an image layer by layer
Registry   → Storage and distribution service for container images

Docker Architecture
───────────────────
Client     → CLI tool that sends commands to the Docker daemon
Daemon     → Background service (dockerd) that manages containers, images, networks
containerd → Industry-standard container runtime managing container lifecycle
runc       → Low-level OCI runtime that creates and runs containers

Networking Models
─────────────────
Bridge   → Default network; isolated network for containers on a single host
Host     → Container shares the host's network namespace directly
Overlay  → Multi-host networking for Docker Swarm or orchestrated environments
None     → No networking; fully isolated container

Storage Options
───────────────
Volumes     → Docker-managed storage, persistent across container restarts
Bind Mounts → Map host filesystem paths directly into the container
tmpfs       → In-memory storage, never written to the host filesystem
```

## 📋 Topics Covered

- **Foundations** — Containers vs VMs, Docker architecture, OCI standards, container ecosystem
- **Images & Dockerfiles** — Build context, layer caching, multi-stage builds, .dockerignore
- **Container Runtime** — Docker Engine, containerd, runc, container lifecycle management
- **Networking** — Bridge, host, overlay, macvlan networks, DNS resolution, port mapping
- **Storage & Volumes** — Volumes, bind mounts, tmpfs, storage drivers, data persistence
- **Docker Compose** — Multi-container applications, service dependencies, compose profiles
- **Registries** — Docker Hub, ECR, ACR, GCR, private registries, image distribution
- **Security** — Image scanning, rootless containers, secrets management, supply chain security
- **Performance** — Resource limits, cgroups, image optimization, build caching strategies
- **Best Practices** — Production patterns, 12-factor apps, health checks, graceful shutdown
- **Anti-Patterns** — Common container mistakes in building, running, and deploying

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to containers?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md) then follow [LEARNING-PATH.md](LEARNING-PATH.md)

**Already familiar with Docker?** → Jump to [05-DOCKER-COMPOSE.md](05-DOCKER-COMPOSE.md) or [07-SECURITY.md](07-SECURITY.md)

**Going to production?** → Review [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) and [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [LEARNING-PATH.md](LEARNING-PATH.md) — progressive exercises from basics to production
