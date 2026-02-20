# Container Registries

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [How Registries Work](#how-registries-work)
   - [Image Distribution Architecture](#image-distribution-architecture)
   - [Image Naming Convention](#image-naming-convention)
3. [Registry Comparison](#registry-comparison)
4. [Docker Hub](#docker-hub)
   - [Public and Private Repositories](#public-and-private-repositories)
   - [Rate Limits](#rate-limits)
   - [Official Images](#official-images)
   - [Automated Builds](#automated-builds)
5. [Amazon ECR](#amazon-ecr)
   - [ECR Setup](#ecr-setup)
   - [ECR Authentication](#ecr-authentication)
   - [Lifecycle Policies](#lifecycle-policies)
   - [Cross-Account Access](#cross-account-access)
6. [Azure ACR](#azure-acr)
   - [ACR Setup](#acr-setup)
   - [ACR Authentication](#acr-authentication)
   - [ACR Tasks](#acr-tasks)
   - [Geo-Replication](#geo-replication)
7. [Google Artifact Registry](#google-artifact-registry)
   - [Artifact Registry Setup](#artifact-registry-setup)
   - [Artifact Registry Authentication](#artifact-registry-authentication)
   - [Vulnerability Scanning](#vulnerability-scanning)
8. [GitHub Container Registry](#github-container-registry)
   - [GHCR Setup](#ghcr-setup)
   - [CI/CD with GitHub Actions](#cicd-with-github-actions)
9. [Self-Hosted Registries](#self-hosted-registries)
   - [Docker Registry](#docker-registry)
   - [Harbor](#harbor)
10. [Image Signing and Verification](#image-signing-and-verification)
    - [Cosign](#cosign)
    - [Notary and Docker Content Trust](#notary-and-docker-content-trust)
11. [Registry Authentication Patterns](#registry-authentication-patterns)
    - [docker login](#docker-login)
    - [Credential Helpers](#credential-helpers)
    - [CI/CD Tokens](#cicd-tokens)
12. [Image Lifecycle and Garbage Collection](#image-lifecycle-and-garbage-collection)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

A **container registry** is a storage and distribution system for container images. Registries allow you to push images built locally or in CI/CD pipelines, store them with versioned tags, and pull them onto any host that needs to run those images.

Understanding registries is essential for any containerized workflow — from a single developer pushing to Docker Hub, to an enterprise running private registries with geo-replication, vulnerability scanning, and image signing.

```
Registry Workflow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Developer / CI Pipeline           Registry            Deployment Target
  ┌──────────────────┐     push    ┌──────────┐  pull   ┌──────────────┐
  │  docker build    │ ──────────▶ │  Image   │ ◀────── │  docker run  │
  │  docker push     │             │  Store   │         │  k8s pod     │
  └──────────────────┘             └──────────┘         └──────────────┘
```

### Target Audience

- **Developers** publishing and consuming container images
- **DevOps Engineers** integrating registries into CI/CD pipelines
- **SREs** managing production image distribution, security, and availability
- **Platform Engineers** selecting and operating registry infrastructure for organizations

### Scope

- Registry concepts — image naming, tags, digests, and manifests
- Major registry services — Docker Hub, ECR, ACR, Artifact Registry, GHCR
- Self-hosted registries — Docker Registry and Harbor
- Image signing, verification, and trust
- Authentication patterns for interactive and automated workflows
- Image lifecycle management and garbage collection

---

## How Registries Work

### Image Distribution Architecture

Registries implement the **OCI Distribution Specification** (formerly Docker Registry HTTP API V2). When you push or pull an image, the Docker client communicates with the registry over HTTPS.

```
Image Push/Pull Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  docker push myregistry.io/app:v1
  ┌───────────────┐         ┌─────────────────┐
  │ Docker Client │         │    Registry     │
  │               │         │                 │
  │ 1. Auth ──────┼────────▶│ Verify token    │
  │               │         │                 │
  │ 2. Push layer ┼────────▶│ Store blob      │
  │    (blob)     │         │ (deduplicated)  │
  │               │         │                 │
  │ 3. Push       ┼────────▶│ Store manifest  │
  │    manifest   │         │ (tag → digest)  │
  └───────────────┘         └─────────────────┘
```

An image is composed of:

| Component | Description |
|---|---|
| **Manifest** | JSON document listing the image layers and configuration |
| **Config blob** | JSON with image metadata (entrypoint, env, labels, architecture) |
| **Layer blobs** | Compressed tar archives of filesystem changes |
| **Tag** | Human-readable pointer to a manifest (e.g., `v1.2.0`, `latest`) |
| **Digest** | Immutable content-addressable hash (e.g., `sha256:abc123...`) |

### Image Naming Convention

```
Full image reference:
  registry.io/namespace/repository:tag@sha256:digest

Examples:
  docker.io/library/nginx:1.25          # Docker Hub official
  docker.io/myuser/myapp:v2             # Docker Hub user
  123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest   # AWS ECR
  myregistry.azurecr.io/myapp:v1        # Azure ACR
  us-docker.pkg.dev/project/repo/app:v1 # Google Artifact Registry
  ghcr.io/myorg/myapp:latest            # GitHub GHCR
```

---

## Registry Comparison

| Feature | Docker Hub | Amazon ECR | Azure ACR | Google AR | GitHub GHCR | Self-Hosted |
|---|---|---|---|---|---|---|
| **Type** | SaaS | Managed | Managed | Managed | SaaS | On-premises |
| **Free tier** | 1 private repo | 500 MB/month | — | 500 MB/month | Unlimited public | Free (self-run) |
| **Private repos** | Paid plans | Unlimited | Unlimited | Unlimited | Unlimited | Unlimited |
| **Vulnerability scan** | Paid | ✅ Built-in | ✅ Defender | ✅ Built-in | ✅ Dependabot | Harbor only |
| **Geo-replication** | CDN | Cross-region | ✅ Premium | Multi-region | CDN | Manual |
| **Image signing** | Content Trust | Signer | Notation | Binary Auth | Sigstore | Harbor/Notary |
| **Rate limits** | 100 pulls/6h (anon) | None | None | None | None | None |
| **CI/CD integration** | Webhooks | IAM roles | Service principals | Workload Identity | Native Actions | API tokens |
| **Max image size** | 10 GB | 42 GB | Unlimited | 5 TB | 10 GB | Configurable |

> **Note:** Pricing and limits change frequently. Check each provider's documentation for current details.

---

## Docker Hub

Docker Hub is the default public registry and the largest library of container images. When you run `docker pull nginx`, Docker pulls from Docker Hub (`docker.io/library/nginx`).

### Public and Private Repositories

```bash
# Tag an image for Docker Hub
docker tag myapp:latest myuser/myapp:v1.0.0

# Push to Docker Hub
docker push myuser/myapp:v1.0.0

# Pull from Docker Hub (explicit)
docker pull docker.io/myuser/myapp:v1.0.0

# Pull from Docker Hub (implicit — default registry)
docker pull myuser/myapp:v1.0.0
```

### Rate Limits

| Account Type | Pull Rate Limit |
|---|---|
| Anonymous | 100 pulls per 6 hours (per IP) |
| Authenticated (free) | 200 pulls per 6 hours |
| Docker Pro/Team/Business | Unlimited |

```bash
# Check remaining pulls
TOKEN=$(curl -s "https://auth.docker.io/token?service=registry.docker.io&scope=repository:library/nginx:pull" | jq -r .token)
curl -s -H "Authorization: Bearer $TOKEN" -D - "https://registry-1.docker.io/v2/library/nginx/manifests/latest" 2>&1 | grep -i ratelimit
```

### Official Images

Official images are curated, security-scanned images maintained by Docker and upstream projects. They live under the `library/` namespace.

```bash
# Official images — no namespace prefix needed
docker pull nginx          # docker.io/library/nginx:latest
docker pull postgres:16    # docker.io/library/postgres:16
docker pull node:20-alpine # docker.io/library/node:20-alpine
docker pull python:3.12    # docker.io/library/python:3.12
```

### Automated Builds

Docker Hub supports automated builds triggered by commits to a linked GitHub or Bitbucket repository.

```bash
# Docker Hub automated build — configured via Docker Hub UI
# Build rules example:
#   Source: /^v([0-9.]+)$/   → Tag: {\1}     (v1.2.3 → 1.2.3)
#   Source: main             → Tag: latest
#   Source: develop          → Tag: develop
```

---

## Amazon ECR

Amazon Elastic Container Registry (ECR) is a fully managed registry integrated with AWS IAM for authentication and authorization.

### ECR Setup

```bash
# Create an ECR repository
aws ecr create-repository \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256 \
  --region us-east-1

# List repositories
aws ecr describe-repositories --region us-east-1

# Get repository URI
aws ecr describe-repositories \
  --repository-names myapp \
  --query 'repositories[0].repositoryUri' \
  --output text
```

### ECR Authentication

```bash
# Login to ECR (token valid for 12 hours)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Tag and push
docker tag myapp:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0.0
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0.0

# Pull
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0.0
```

### Lifecycle Policies

Lifecycle policies automate image cleanup to manage storage costs.

```bash
# Create a lifecycle policy
aws ecr put-lifecycle-policy \
  --repository-name myapp \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Remove untagged images older than 7 days",
        "selection": {
          "tagStatus": "untagged",
          "countType": "sinceImagePushed",
          "countUnit": "days",
          "countNumber": 7
        },
        "action": { "type": "expire" }
      },
      {
        "rulePriority": 2,
        "description": "Keep only the last 20 tagged images",
        "selection": {
          "tagStatus": "tagged",
          "tagPrefixList": ["v"],
          "countType": "imageCountMoreThan",
          "countNumber": 20
        },
        "action": { "type": "expire" }
      }
    ]
  }'
```

### Cross-Account Access

```bash
# Grant another AWS account pull access
aws ecr set-repository-policy \
  --repository-name myapp \
  --policy-text '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowCrossAccountPull",
        "Effect": "Allow",
        "Principal": {
          "AWS": "arn:aws:iam::987654321098:root"
        },
        "Action": [
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:BatchCheckLayerAvailability"
        ]
      }
    ]
  }'
```

---

## Azure ACR

Azure Container Registry (ACR) is a managed registry service integrated with Azure Active Directory and Azure services.

### ACR Setup

```bash
# Create a resource group
az group create --name myapp-rg --location eastus

# Create a container registry (Basic, Standard, or Premium)
az acr create \
  --resource-group myapp-rg \
  --name myappregistry \
  --sku Standard \
  --admin-enabled false

# List registries
az acr list --resource-group myapp-rg --output table

# Get login server
az acr show --name myappregistry --query loginServer --output tsv
```

### ACR Authentication

```bash
# Login with Azure CLI (uses Azure AD token)
az acr login --name myappregistry

# Tag and push
docker tag myapp:latest myappregistry.azurecr.io/myapp:v1.0.0
docker push myappregistry.azurecr.io/myapp:v1.0.0

# Pull
docker pull myappregistry.azurecr.io/myapp:v1.0.0

# Create a service principal for CI/CD
ACR_ID=$(az acr show --name myappregistry --query id --output tsv)
az ad sp create-for-rbac \
  --name myapp-ci-push \
  --scopes $ACR_ID \
  --role acrpush
```

### ACR Tasks

ACR Tasks allow you to build images directly in the cloud without a local Docker daemon.

```bash
# Quick build in the cloud
az acr build \
  --registry myappregistry \
  --image myapp:v1.0.0 \
  --file Dockerfile \
  .

# Multi-step task from a task definition
az acr task create \
  --registry myappregistry \
  --name build-and-test \
  --file acr-task.yaml \
  --context https://github.com/myorg/myapp.git \
  --git-access-token $GITHUB_TOKEN

# Run a task manually
az acr task run --registry myappregistry --name build-and-test
```

### Geo-Replication

Geo-replication (Premium tier) replicates images to multiple Azure regions for low-latency pulls.

```bash
# Enable geo-replication (Premium SKU required)
az acr replication create \
  --registry myappregistry \
  --location westeurope

az acr replication create \
  --registry myappregistry \
  --location southeastasia

# List replications
az acr replication list --registry myappregistry --output table
```

---

## Google Artifact Registry

Google Artifact Registry is the recommended container registry for Google Cloud, replacing the legacy Container Registry (GCR).

### Artifact Registry Setup

```bash
# Enable the Artifact Registry API
gcloud services enable artifactregistry.googleapis.com

# Create a Docker repository
gcloud artifacts repositories create myapp-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="My application images"

# List repositories
gcloud artifacts repositories list --location=us-central1
```

### Artifact Registry Authentication

```bash
# Configure Docker credential helper
gcloud auth configure-docker us-central1-docker.pkg.dev

# Tag and push
docker tag myapp:latest us-central1-docker.pkg.dev/my-project/myapp-repo/myapp:v1.0.0
docker push us-central1-docker.pkg.dev/my-project/myapp-repo/myapp:v1.0.0

# Pull
docker pull us-central1-docker.pkg.dev/my-project/myapp-repo/myapp:v1.0.0

# Service account authentication for CI/CD
gcloud auth activate-service-account \
  --key-file=service-account-key.json
gcloud auth configure-docker us-central1-docker.pkg.dev
```

### Vulnerability Scanning

```bash
# Enable vulnerability scanning (Container Analysis API)
gcloud services enable containeranalysis.googleapis.com

# Check scan results for an image
gcloud artifacts docker images list \
  us-central1-docker.pkg.dev/my-project/myapp-repo/myapp \
  --show-occurrences \
  --format=json

# Describe vulnerabilities for a specific image
gcloud artifacts docker images describe \
  us-central1-docker.pkg.dev/my-project/myapp-repo/myapp:v1.0.0 \
  --show-package-vulnerability
```

---

## GitHub Container Registry

GitHub Container Registry (GHCR) is integrated with GitHub and GitHub Actions, making it ideal for open-source projects and GitHub-centric workflows.

### GHCR Setup

```bash
# Login to GHCR with a personal access token
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Tag and push
docker tag myapp:latest ghcr.io/myorg/myapp:v1.0.0
docker push ghcr.io/myorg/myapp:v1.0.0

# Pull a public image (no auth needed)
docker pull ghcr.io/myorg/myapp:v1.0.0
```

### CI/CD with GitHub Actions

```yaml
# .github/workflows/publish.yml
name: Build and Publish

on:
  push:
    tags:
      - 'v*'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

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
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

---

## Self-Hosted Registries

Self-hosted registries give you full control over image storage, access policies, and network location.

### Docker Registry

The official Docker Registry is a lightweight, open-source registry server.

```yaml
# compose.yaml — self-hosted Docker Registry
services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
    volumes:
      - registry-data:/var/lib/registry
      - ./certs:/certs:ro
      - ./auth:/auth:ro
    environment:
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
      REGISTRY_HTTP_TLS_KEY: /certs/domain.key
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: "Registry Realm"
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      REGISTRY_STORAGE_DELETE_ENABLED: "true"
    restart: unless-stopped

volumes:
  registry-data:
```

```bash
# Generate htpasswd file for basic auth
docker run --rm --entrypoint htpasswd httpd:2 -Bbn admin secretpassword > auth/htpasswd

# Generate self-signed TLS certificate
openssl req -newkey rsa:4096 -nodes -sha256 \
  -keyout certs/domain.key -x509 -days 365 \
  -out certs/domain.crt \
  -subj "/CN=registry.example.com"

# Start the registry
docker compose up -d

# Push to the self-hosted registry
docker tag myapp:latest registry.example.com:5000/myapp:v1.0.0
docker push registry.example.com:5000/myapp:v1.0.0
```

### Harbor

Harbor is an open-source enterprise registry with vulnerability scanning, RBAC, image signing, and replication.

```yaml
# Harbor installation with the online installer
# Download from https://github.com/goharbor/harbor/releases
```

```bash
# Download and extract Harbor installer
curl -sL https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-online-installer-v2.10.0.tgz | tar xz
cd harbor

# Edit harbor.yml configuration
# Set hostname, HTTPS certificates, admin password, database

# Run the installer
./install.sh --with-trivy       # Include Trivy vulnerability scanner
```

```
Harbor Architecture
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────────────────────────────────┐
  │              Harbor Portal               │
  │           (Web UI + REST API)            │
  └─────────┬────────────┬──────────────────┘
            │            │
  ┌─────────▼───┐  ┌─────▼──────────┐
  │  Registry   │  │  Trivy Scanner │
  │  (storage)  │  │  (CVE scan)    │
  └─────────────┘  └────────────────┘
            │
  ┌─────────▼───┐  ┌────────────────┐
  │  PostgreSQL │  │  Redis Cache   │
  │  (metadata) │  │  (sessions)    │
  └─────────────┘  └────────────────┘
```

---

## Image Signing and Verification

Image signing ensures that images have not been tampered with between build and deployment.

### Cosign

Cosign (part of the Sigstore project) is the modern standard for container image signing.

```bash
# Install cosign
go install github.com/sigstore/cosign/v2/cmd/cosign@latest

# Generate a key pair
cosign generate-key-pair

# Sign an image
cosign sign --key cosign.key myregistry.io/myapp:v1.0.0

# Verify an image signature
cosign verify --key cosign.pub myregistry.io/myapp:v1.0.0

# Keyless signing with OIDC (Sigstore Fulcio + Rekor)
cosign sign myregistry.io/myapp:v1.0.0
# Opens browser for OIDC authentication, stores signature in Rekor transparency log

# Verify keyless signature
cosign verify \
  --certificate-identity=user@example.com \
  --certificate-oidc-issuer=https://accounts.google.com \
  myregistry.io/myapp:v1.0.0
```

### Notary and Docker Content Trust

Docker Content Trust (DCT) uses Notary to sign and verify image tags.

```bash
# Enable Docker Content Trust
export DOCKER_CONTENT_TRUST=1

# Push a signed image (prompts for passphrase on first use)
docker push myregistry.io/myapp:v1.0.0

# Pull only signed images
docker pull myregistry.io/myapp:v1.0.0
# Fails if the image is not signed

# Disable DCT for a single command
DOCKER_CONTENT_TRUST=0 docker pull unsigned-image:latest

# Inspect trust data
docker trust inspect --pretty myregistry.io/myapp
```

---

## Registry Authentication Patterns

### docker login

```bash
# Interactive login
docker login myregistry.io

# Non-interactive login (CI/CD)
echo $REGISTRY_PASSWORD | docker login myregistry.io -u $REGISTRY_USER --password-stdin

# Login to multiple registries
docker login docker.io
docker login ghcr.io
docker login myregistry.azurecr.io

# Logout
docker logout myregistry.io
```

> **Important:** Avoid passing passwords via the `--password` flag — it may be stored in shell history. Always use `--password-stdin`.

### Credential Helpers

Credential helpers store Docker credentials in the OS keychain or cloud-native credential stores.

```json
// ~/.docker/config.json — credential helper configuration
{
  "credHelpers": {
    "123456789012.dkr.ecr.us-east-1.amazonaws.com": "ecr-login",
    "myregistry.azurecr.io": "acr-env",
    "us-central1-docker.pkg.dev": "gcloud",
    "ghcr.io": "gh"
  }
}
```

| Helper | Registry | Install |
|---|---|---|
| `docker-credential-ecr-login` | Amazon ECR | `go install github.com/awslabs/amazon-ecr-credential-helper/...` |
| `docker-credential-acr-env` | Azure ACR | Included with Azure CLI |
| `docker-credential-gcloud` | Google AR | Included with gcloud CLI |
| `docker-credential-gh` | GitHub GHCR | Included with GitHub CLI |
| `docker-credential-osxkeychain` | Any | Included with Docker Desktop (macOS) |
| `docker-credential-secretservice` | Any | Included with Docker on Linux (GNOME) |

### CI/CD Tokens

```yaml
# GitHub Actions — GITHUB_TOKEN for GHCR
- name: Login to GHCR
  run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

# GitLab CI — CI_REGISTRY variables
docker-build:
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

---

## Image Lifecycle and Garbage Collection

Managing the lifecycle of images in a registry prevents unbounded storage growth and keeps repositories clean.

```
Image Lifecycle
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Build ──▶ Push ──▶ Tag ──▶ Deploy ──▶ Retire
                      │                    │
                      │   Retention        │
                      │   Policy           │
                      ▼                    ▼
                Keep N latest        Delete old
                Keep by tag          untagged
                Keep by date         manifests
```

```bash
# Docker Hub — manage tags via CLI
# Delete a tag using the Hub API
HUB_TOKEN=$(curl -s -X POST "https://hub.docker.com/v2/users/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"user","password":"pass"}' | jq -r .token)

curl -s -X DELETE \
  -H "Authorization: Bearer $HUB_TOKEN" \
  "https://hub.docker.com/v2/repositories/myuser/myapp/tags/old-tag/"

# Self-hosted registry — garbage collection
docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml

# Self-hosted registry — garbage collection (dry run)
docker exec registry bin/registry garbage-collect --dry-run /etc/docker/registry/config.yml
```

Recommended tag retention strategies:

| Strategy | Description | Example |
|---|---|---|
| **Keep N latest** | Retain the N most recently pushed tags | Keep last 20 tagged images |
| **Keep by age** | Delete images older than a threshold | Remove untagged images > 7 days |
| **Keep by pattern** | Retain tags matching a regex | Keep `v*` tags, delete `dev-*` after 30 days |
| **Keep by usage** | Retain images actively pulled | Delete images not pulled in 90 days |

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
| [05-DOCKER-COMPOSE.md](05-DOCKER-COMPOSE.md) | Docker Compose | Multi-container applications with Compose files |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Container Registries documentation |
