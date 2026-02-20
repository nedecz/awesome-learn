# Docker Compose

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Compose File Structure](#compose-file-structure)
   - [Top-Level Elements](#top-level-elements)
   - [Complete Example](#complete-example)
3. [Service Configuration](#service-configuration)
   - [image](#image)
   - [build](#build)
   - [ports](#ports)
   - [volumes](#volumes)
   - [environment](#environment)
   - [depends_on](#depends_on)
   - [healthcheck](#healthcheck)
   - [restart](#restart)
4. [Networking in Compose](#networking-in-compose)
   - [Default Network](#default-network)
   - [Custom Networks](#custom-networks)
   - [Network Aliases](#network-aliases)
5. [Volume Management in Compose](#volume-management-in-compose)
   - [Named Volumes](#named-volumes)
   - [Bind Mounts in Compose](#bind-mounts-in-compose)
6. [Environment Variables and .env Files](#environment-variables-and-env-files)
   - [Inline Variables](#inline-variables)
   - [env_file Directive](#env_file-directive)
   - [Variable Substitution](#variable-substitution)
7. [Compose Profiles](#compose-profiles)
8. [Docker Compose CLI Commands](#docker-compose-cli-commands)
   - [up](#up)
   - [down](#down)
   - [ps](#ps)
   - [logs](#logs)
   - [exec](#exec)
   - [build](#build-command)
   - [pull](#pull)
   - [restart](#restart-command)
9. [Multi-Environment Setup](#multi-environment-setup)
   - [Override Files](#override-files)
   - [Development Configuration](#development-configuration)
   - [Production Configuration](#production-configuration)
10. [Compose Watch](#compose-watch)
11. [Common Patterns](#common-patterns)
    - [Web App + Database + Cache](#web-app--database--cache)
    - [Microservices Stack](#microservices-stack)
12. [Health Checks and Dependencies](#health-checks-and-dependencies)
    - [depends_on with Condition](#depends_on-with-condition)
    - [Startup Order Pattern](#startup-order-pattern)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

**Docker Compose** is a tool for defining and running multi-container Docker applications. With a single YAML file, you declare all the services, networks, and volumes your application needs, and then bring the entire stack up or down with a single command.

Compose solves the problem of managing multiple `docker run` commands with complex flags. Instead of running containers individually, you describe the desired state of your entire application stack in a `compose.yaml` (or `docker-compose.yml`) file and let Compose handle the orchestration.

```
Docker Compose Workflow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  compose.yaml              docker compose up
  ┌──────────────┐          ┌──────────────────┐
  │ services:    │          │  Creates:        │
  │   web:       │ ──────▶  │  - Networks      │
  │   db:        │          │  - Volumes       │
  │   cache:     │          │  - Containers    │
  │ networks:    │          │  - Connects them │
  │ volumes:     │          └──────────────────┘
  └──────────────┘
```

### Target Audience

- **Developers** building and testing multi-container applications locally
- **DevOps Engineers** defining reproducible application stacks for CI/CD
- **SREs** managing development and staging environments with consistent configurations
- **Platform Engineers** creating standard application templates for teams

### Scope

- Compose file syntax and all major configuration options
- Service, network, and volume definitions
- Environment variable management and multi-environment setups
- CLI commands for managing the full application lifecycle
- Common patterns for web applications, databases, and microservices
- Compose Watch for development hot-reload workflows

---

## Compose File Structure

Compose uses a YAML file to configure application services. The default filename is `compose.yaml` (recommended) or `docker-compose.yml` (legacy but still supported).

### Top-Level Elements

```yaml
# compose.yaml — top-level structure
name: my-application

services:
  # Define each container as a service
  web:
    image: nginx:latest
  api:
    build: ./api

networks:
  # Define custom networks
  frontend:
  backend:

volumes:
  # Define named volumes
  db-data:
  cache-data:

configs:
  # Define configuration objects
  nginx-config:
    file: ./nginx.conf

secrets:
  # Define secret objects
  db-password:
    file: ./secrets/db-password.txt
```

| Element | Purpose | Required |
|---|---|---|
| `name` | Project name (defaults to directory name) | No |
| `services` | Container definitions — the core of every Compose file | Yes |
| `networks` | Custom network definitions for inter-service communication | No |
| `volumes` | Named volume definitions for persistent data | No |
| `configs` | Configuration objects mounted into services | No |
| `secrets` | Sensitive data mounted into services | No |

### Complete Example

```yaml
# compose.yaml — full-stack web application
name: fullstack-app

services:
  web:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - static-assets:/usr/share/nginx/html:ro
    depends_on:
      api:
        condition: service_healthy
    networks:
      - frontend

  api:
    build:
      context: ./api
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/myapp
      - REDIS_URL=redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - frontend
      - backend

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks:
      - backend

  cache:
    image: redis:7-alpine
    volumes:
      - redisdata:/data
    networks:
      - backend

networks:
  frontend:
  backend:

volumes:
  pgdata:
  redisdata:
  static-assets:
```

---

## Service Configuration

Each key under `services:` defines a container. The following sections cover the most commonly used service options.

### image

Specifies the Docker image to start the container from.

```yaml
services:
  web:
    image: nginx:latest           # Docker Hub official image
  api:
    image: myregistry.io/api:v2   # Private registry image
  db:
    image: postgres:16-alpine     # Specific tag
```

### build

Builds an image from a Dockerfile instead of pulling a pre-built image.

```yaml
services:
  # Simple build from context directory
  api:
    build: ./api

  # Build with explicit options
  worker:
    build:
      context: ./worker
      dockerfile: Dockerfile.prod
      args:
        NODE_ENV: production
        APP_VERSION: "2.1.0"
      target: runtime
      cache_from:
        - myregistry.io/worker:cache
```

### ports

Maps container ports to the host.

```yaml
services:
  web:
    ports:
      - "80:80"                   # HOST:CONTAINER
      - "443:443"
      - "8080:80"                 # Map host 8080 to container 80
      - "127.0.0.1:3000:3000"    # Bind to localhost only
      - "9090-9091:8080-8081"    # Port range
```

### volumes

Mounts volumes or bind mounts into the container.

```yaml
services:
  api:
    volumes:
      - pgdata:/var/lib/postgresql/data      # Named volume
      - ./src:/app/src                        # Bind mount (relative path)
      - /etc/localtime:/etc/localtime:ro      # Host file (read-only)
      - type: tmpfs                           # tmpfs mount
        target: /tmp
        tmpfs:
          size: 64m
```

### environment

Sets environment variables in the container.

```yaml
services:
  api:
    # Map syntax
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/myapp
      LOG_LEVEL: info
      DEBUG: "false"

  worker:
    # List syntax
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/myapp
      - LOG_LEVEL=info
      - DEBUG=false
```

### depends_on

Defines startup and shutdown dependencies between services.

```yaml
services:
  api:
    depends_on:
      - db        # Simple form — starts db before api
      - cache

  web:
    depends_on:
      api:
        condition: service_healthy    # Wait for health check
      db:
        condition: service_started    # Wait for container start
```

### healthcheck

Defines a command to check whether the service is healthy.

```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s       # Time between checks
      timeout: 5s         # Max time for a single check
      retries: 5          # Failures before marking unhealthy
      start_period: 30s   # Grace period during startup

  db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 10
```

### restart

Defines the restart policy for the service.

```yaml
services:
  api:
    restart: unless-stopped    # Restart unless explicitly stopped

  worker:
    restart: always            # Always restart on exit

  migration:
    restart: "no"              # Never restart (default)

  batch-job:
    restart: on-failure        # Restart only on non-zero exit code
```

| Policy | Behavior |
|---|---|
| `no` | Never restart automatically (default) |
| `always` | Always restart when the process exits |
| `on-failure` | Restart only when the process exits with a non-zero code |
| `unless-stopped` | Like `always`, but does not restart after manual `docker stop` |

---

## Networking in Compose

### Default Network

Compose automatically creates a **default network** for your project. All services join this network and can reach each other by **service name**.

```yaml
# compose.yaml
services:
  api:
    image: myapp/api:latest
    # Can connect to "db" by hostname
  db:
    image: postgres:16
    # Can connect to "api" by hostname
```

```
Default Network
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────────────────────────┐
  │     myapp_default (bridge)      │
  │                                 │
  │  ┌─────────┐   ┌─────────┐     │
  │  │   api   │◀─▶│   db    │     │
  │  │ :3000   │   │ :5432   │     │
  │  └─────────┘   └─────────┘     │
  └─────────────────────────────────┘

  api can reach db at hostname "db"
  db can reach api at hostname "api"
```

### Custom Networks

Custom networks isolate groups of services and control which services can communicate with each other.

```yaml
services:
  web:
    networks:
      - frontend

  api:
    networks:
      - frontend
      - backend

  db:
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true    # No external access
```

```
Custom Network Isolation
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────────────────┐
  │     frontend (bridge)    │
  │  ┌──────┐   ┌──────┐    │
  │  │ web  │──▶│ api  │    │
  │  └──────┘   └──┬───┘    │
  └────────────────┼────────┘
                   │
  ┌────────────────┼────────┐
  │     backend (internal)  │
  │            ┌───▼──┐     │
  │            │  db  │     │
  │            └──────┘     │
  └─────────────────────────┘

  web ──▶ api  ✅ (both on frontend)
  api ──▶ db   ✅ (both on backend)
  web ──▶ db   ❌ (different networks)
```

### Network Aliases

Network aliases give a service additional hostnames on a specific network.

```yaml
services:
  primary-db:
    image: postgres:16
    networks:
      backend:
        aliases:
          - database
          - postgres
          - db

  replica-db:
    image: postgres:16
    networks:
      backend:
        aliases:
          - database-replica
          - postgres-ro
```

---

## Volume Management in Compose

### Named Volumes

Named volumes are defined in the top-level `volumes` section and referenced by services.

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data

  backup:
    image: alpine
    volumes:
      - pgdata:/data:ro    # Same volume, read-only

volumes:
  pgdata:                   # Default local driver
  cache-data:
    driver: local
  nfs-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw,nfsvers=4
      device: ":/exports/data"
```

### Bind Mounts in Compose

```yaml
services:
  api:
    build: ./api
    volumes:
      - ./api/src:/app/src            # Source code
      - ./api/package.json:/app/package.json:ro
      - /var/run/docker.sock:/var/run/docker.sock  # Docker socket
```

> **Note:** Bind mount paths in Compose are relative to the directory containing the `compose.yaml` file, not the current working directory.

---

## Environment Variables and .env Files

### Inline Variables

```yaml
services:
  api:
    environment:
      NODE_ENV: production
      PORT: "3000"
      DATABASE_URL: postgres://user:pass@db:5432/myapp
```

### env_file Directive

```yaml
services:
  api:
    env_file:
      - .env                    # Default environment
      - .env.local              # Local overrides (not committed)

  worker:
    env_file:
      - path: .env.production
        required: true
      - path: .env.local
        required: false          # Optional — no error if missing
```

```bash
# .env file format
DATABASE_URL=postgres://user:pass@db:5432/myapp
REDIS_URL=redis://cache:6379
LOG_LEVEL=info
SECRET_KEY=supersecretkey123
```

### Variable Substitution

Compose supports shell-style variable substitution in the YAML file.

```yaml
services:
  api:
    image: myregistry.io/api:${APP_VERSION:-latest}
    environment:
      DATABASE_URL: postgres://${DB_USER}:${DB_PASS}@db:5432/${DB_NAME}
    ports:
      - "${HOST_PORT:-3000}:3000"
```

| Syntax | Behavior |
|---|---|
| `${VAR}` | Value of `VAR`, error if unset |
| `${VAR:-default}` | Value of `VAR`, or `default` if unset or empty |
| `${VAR-default}` | Value of `VAR`, or `default` if unset (empty is kept) |
| `${VAR:?error}` | Value of `VAR`, or exit with `error` if unset or empty |
| `${VAR?error}` | Value of `VAR`, or exit with `error` if unset |

```bash
# .env file used for variable substitution
APP_VERSION=2.1.0
DB_USER=myapp
DB_PASS=secret
DB_NAME=production
HOST_PORT=8080
```

---

## Compose Profiles

Profiles allow you to selectively start services. Services without a `profiles` attribute start by default. Services with a `profiles` attribute only start when that profile is explicitly activated.

```yaml
services:
  web:
    image: nginx:latest
    # No profiles — always starts

  api:
    build: ./api
    # No profiles — always starts

  db:
    image: postgres:16
    # No profiles — always starts

  debug:
    image: busybox
    profiles:
      - debug
    # Only starts with --profile debug

  monitoring:
    image: prom/prometheus:latest
    profiles:
      - monitoring
      - debug
    # Starts with --profile monitoring OR --profile debug

  seed:
    image: myapp/seed:latest
    profiles:
      - setup
    # Only starts with --profile setup
```

```bash
# Start default services only
docker compose up -d

# Start default services + debug profile
docker compose --profile debug up -d

# Start default services + multiple profiles
docker compose --profile monitoring --profile debug up -d

# Run a one-off service from a profile
docker compose run --rm seed
```

---

## Docker Compose CLI Commands

### up

Creates and starts all services defined in the Compose file.

```bash
# Start all services in detached mode
docker compose up -d

# Start and rebuild images
docker compose up -d --build

# Start specific services
docker compose up -d web api

# Start with scaled replicas
docker compose up -d --scale worker=3

# Start and wait for health checks
docker compose up -d --wait
```

### down

Stops and removes containers, networks, and optionally volumes and images.

```bash
# Stop and remove containers and networks
docker compose down

# Also remove named volumes
docker compose down -v

# Also remove images
docker compose down --rmi all

# Remove orphan containers (from removed services)
docker compose down --remove-orphans
```

### ps

Lists containers managed by Compose.

```bash
# List running containers
docker compose ps

# List all containers (including stopped)
docker compose ps -a

# List in quiet mode (IDs only)
docker compose ps -q
```

### logs

Views log output from services.

```bash
# View logs from all services
docker compose logs

# Follow logs in real time
docker compose logs -f

# View logs from specific services
docker compose logs -f api db

# Show last 100 lines
docker compose logs --tail 100

# Show logs with timestamps
docker compose logs -t
```

### exec

Executes a command in a running service container.

```bash
# Open a shell in a service
docker compose exec api sh

# Run a one-off command
docker compose exec db psql -U postgres -d myapp

# Run as a specific user
docker compose exec --user root api bash

# Set environment variables
docker compose exec -e DEBUG=true api node script.js
```

### build {#build-command}

Builds or rebuilds service images.

```bash
# Build all services with build configuration
docker compose build

# Build specific services
docker compose build api worker

# Build without cache
docker compose build --no-cache

# Build with build arguments
docker compose build --build-arg NODE_ENV=production
```

### pull

Pulls service images from the registry.

```bash
# Pull all images
docker compose pull

# Pull specific service images
docker compose pull web db

# Pull ignoring services that have build configuration
docker compose pull --ignore-buildable
```

### restart {#restart-command}

Restarts running services.

```bash
# Restart all services
docker compose restart

# Restart specific services
docker compose restart api worker

# Restart with a custom timeout
docker compose restart -t 30 api
```

---

## Multi-Environment Setup

### Override Files

Compose supports layering multiple files. Properties in later files override earlier ones.

```
Override File Resolution
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  compose.yaml          Base configuration
       │
       ▼
  compose.override.yaml Auto-loaded override
       │                (development defaults)
       ▼
  compose.prod.yaml     Explicit override
                        (production settings)
```

```bash
# Default: loads compose.yaml + compose.override.yaml
docker compose up -d

# Explicit: loads compose.yaml + compose.prod.yaml
docker compose -f compose.yaml -f compose.prod.yaml up -d

# Multiple overrides
docker compose -f compose.yaml -f compose.staging.yaml -f compose.monitoring.yaml up -d
```

### Development Configuration

```yaml
# compose.yaml — base configuration
services:
  api:
    build: ./api
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/myapp
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  pgdata:
```

```yaml
# compose.override.yaml — development overrides (auto-loaded)
services:
  api:
    build:
      context: ./api
      target: development
    volumes:
      - ./api/src:/app/src
    ports:
      - "3000:3000"
      - "9229:9229"     # Node.js debugger
    environment:
      LOG_LEVEL: debug
      NODE_ENV: development

  db:
    ports:
      - "5432:5432"     # Expose DB for local tools
```

### Production Configuration

```yaml
# compose.prod.yaml — production overrides
services:
  api:
    build:
      context: ./api
      target: production
    restart: unless-stopped
    environment:
      LOG_LEVEL: warn
      NODE_ENV: production
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M

  db:
    restart: unless-stopped
    volumes:
      - pgdata:/var/lib/postgresql/data
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 1G
```

---

## Compose Watch

Compose Watch automatically updates running services when source files change. It is the recommended approach for development hot-reload with Docker Compose.

```yaml
# compose.yaml with watch configuration
services:
  api:
    build: ./api
    ports:
      - "3000:3000"
    develop:
      watch:
        # Sync source files into the container
        - action: sync
          path: ./api/src
          target: /app/src

        # Rebuild when dependencies change
        - action: rebuild
          path: ./api/package.json

        # Sync and restart when config changes
        - action: sync+restart
          path: ./api/config
          target: /app/config
```

| Action | Behavior | Use Case |
|---|---|---|
| `sync` | Copies changed files into the container | Source code with hot-reload (nodemon, webpack-dev-server) |
| `rebuild` | Rebuilds and replaces the container | Dependency changes (package.json, requirements.txt) |
| `sync+restart` | Copies files then restarts the container | Config files that require process restart |

```bash
# Start services with file watching
docker compose watch

# Start in detached mode with watch
docker compose up -d && docker compose watch
```

---

## Common Patterns

### Web App + Database + Cache

A typical three-tier web application with nginx, a backend API, PostgreSQL, and Redis.

```yaml
# compose.yaml — three-tier web application
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      api:
        condition: service_healthy
    networks:
      - frontend

  api:
    build: ./api
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/webapp
      REDIS_URL: redis://cache:6379
      PORT: "3000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    networks:
      - frontend
      - backend

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: webapp
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d webapp"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks:
      - backend

  cache:
    image: redis:7-alpine
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru
    volumes:
      - redisdata:/data
    networks:
      - backend

networks:
  frontend:
  backend:
    internal: true

volumes:
  pgdata:
  redisdata:
```

### Microservices Stack

A microservices architecture with independent services, a message queue, and an API gateway.

```yaml
# compose.yaml — microservices stack
services:
  gateway:
    build: ./gateway
    ports:
      - "8080:8080"
    environment:
      USER_SERVICE_URL: http://user-service:3001
      ORDER_SERVICE_URL: http://order-service:3002
      PRODUCT_SERVICE_URL: http://product-service:3003
    depends_on:
      - user-service
      - order-service
      - product-service
    networks:
      - public
      - internal

  user-service:
    build: ./services/user
    environment:
      DATABASE_URL: postgres://user:pass@user-db:5432/users
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672
    depends_on:
      user-db:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    networks:
      - internal

  order-service:
    build: ./services/order
    environment:
      DATABASE_URL: postgres://user:pass@order-db:5432/orders
      RABBITMQ_URL: amqp://guest:guest@rabbitmq:5672
    depends_on:
      order-db:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    networks:
      - internal

  product-service:
    build: ./services/product
    environment:
      DATABASE_URL: postgres://user:pass@product-db:5432/products
    depends_on:
      product-db:
        condition: service_healthy
    networks:
      - internal

  user-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: users
    volumes:
      - user-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks:
      - internal

  order-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: orders
    volumes:
      - order-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks:
      - internal

  product-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: products
    volumes:
      - product-db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks:
      - internal

  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "15672:15672"    # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_running"]
      interval: 10s
      timeout: 5s
      retries: 10
    networks:
      - internal

networks:
  public:
  internal:
    internal: true

volumes:
  user-db-data:
  order-db-data:
  product-db-data:
  rabbitmq-data:
```

---

## Health Checks and Dependencies

### depends_on with Condition

The `depends_on` directive with `condition` ensures services only start after their dependencies reach a specific state.

```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy     # Wait for health check to pass
        restart: true                  # Restart api if db restarts
      cache:
        condition: service_started     # Wait for container to start
      migration:
        condition: service_completed_successfully  # Wait for exit code 0
```

| Condition | Behavior |
|---|---|
| `service_started` | Waits for the dependency container to start (default) |
| `service_healthy` | Waits for the dependency health check to report healthy |
| `service_completed_successfully` | Waits for the dependency to exit with code 0 |

### Startup Order Pattern

A common pattern for applications that need database migrations before the API starts.

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 10

  migration:
    build: ./api
    command: ["npm", "run", "migrate"]
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/myapp
    depends_on:
      db:
        condition: service_healthy

  api:
    build: ./api
    command: ["npm", "start"]
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/myapp
    depends_on:
      migration:
        condition: service_completed_successfully
```

```
Startup Order
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────┐
  │    db    │ ── starts first
  └────┬─────┘
       │ healthy ✅
       ▼
  ┌──────────┐
  │migration │ ── runs migrations
  └────┬─────┘
       │ exit 0 ✅
       ▼
  ┌──────────┐
  │   api    │ ── starts serving
  └──────────┘
```

---

## Next Steps

Continue your Docker learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Docker Overview | Core Docker concepts, architecture, and CLI essentials |
| [01-IMAGES-AND-DOCKERFILES.md](01-IMAGES-AND-DOCKERFILES.md) | Images & Dockerfiles | Building images, Dockerfile instructions, multi-stage builds |
| [02-CONTAINER-RUNTIME.md](02-CONTAINER-RUNTIME.md) | Container Runtime | Docker Engine, containerd, runc, and container lifecycle internals |
| [03-NETWORKING.md](03-NETWORKING.md) | Docker Networking | Network drivers, DNS discovery, port mapping, and network security |
| [04-STORAGE-AND-VOLUMES.md](04-STORAGE-AND-VOLUMES.md) | Storage & Volumes | Volumes, bind mounts, tmpfs, and storage drivers |
| [06-REGISTRIES.md](06-REGISTRIES.md) | Container Registries | Image distribution, registry services, and signing |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Docker Compose documentation |
