# Microservices Deployment Strategies

## Table of Contents

1. [Overview](#overview)
2. [Rolling Updates](#rolling-updates)
3. [Blue-Green Deployment](#blue-green-deployment)
4. [Canary Deployment](#canary-deployment)
5. [Feature Flags](#feature-flags)
6. [GitOps](#gitops)
7. [CI/CD for Microservices](#cicd-for-microservices)
8. [Containerization](#containerization)
9. [Choosing a Strategy](#choosing-a-strategy)
10. [Best Practices](#best-practices)
11. [Next Steps](#next-steps)

## Overview

Deploying microservices requires strategies that minimize risk, enable fast rollbacks, and support independent service releases. This document covers the deployment patterns and practices used in production microservices environments.

### Target Audience

- DevOps engineers building deployment pipelines
- SREs managing production rollouts
- Developers responsible for releasing their services

### Scope

- Rolling update, blue-green, and canary deployment patterns
- Feature flag management for progressive delivery
- GitOps with ArgoCD and Flux
- CI/CD pipeline design for microservices
- Container image best practices

## Rolling Updates

Rolling updates replace instances of the old version with the new version incrementally. At any point during the rollout, both old and new versions are running simultaneously.

```
Rolling Update Process:

Time 0:  [v1] [v1] [v1] [v1]     All instances are v1

Time 1:  [v2] [v1] [v1] [v1]     First instance updated to v2

Time 2:  [v2] [v2] [v1] [v1]     Second instance updated

Time 3:  [v2] [v2] [v2] [v1]     Third instance updated

Time 4:  [v2] [v2] [v2] [v2]     Rollout complete — all v2
```

### Kubernetes Rolling Update

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Allow 1 extra Pod during rollout
      maxUnavailable: 0  # Never reduce below desired count
  template:
    spec:
      containers:
        - name: order-service
          image: order-service:2.1.0
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
```

### Pros and Cons

| Pros | Cons |
|---|---|
| Zero downtime | Both versions run simultaneously (compatibility required) |
| Built into Kubernetes | Rollback is a new rollout (takes time) |
| Simple to configure | Hard to test with a small percentage of traffic |

## Blue-Green Deployment

Blue-green deployment runs two identical environments. One (blue) serves production traffic while the other (green) is updated. Traffic is switched atomically from blue to green.

```
Blue-Green Deployment:

                ┌──────────────────┐
                │   Load Balancer  │
                └────────┬─────────┘
                         │
            ┌────────────┼────────────┐
            │            │            │
     ┌──────▼─────┐           ┌──────▼──────┐
     │ Blue (v1)  │           │ Green (v2)  │
     │ ┌────┐┌────┐│           │ ┌────┐┌────┐│
     │ │ v1 ││ v1 ││ ◀─ LIVE  │ │ v2 ││ v2 ││ ◀─ STAGING
     │ └────┘└────┘│           │ └────┘└────┘│
     └─────────────┘           └─────────────┘

After switch:

     ┌─────────────┐           ┌─────────────┐
     │ Blue (v1)   │           │ Green (v2)  │
     │ ┌────┐┌────┐│           │ ┌────┐┌────┐│
     │ │ v1 ││ v1 ││ ◀─ IDLE  │ │ v2 ││ v2 ││ ◀─ LIVE
     │ └────┘└────┘│           │ └────┘└────┘│
     └─────────────┘           └─────────────┘
```

### Pros and Cons

| Pros | Cons |
|---|---|
| Instant rollback (switch back to blue) | Requires double the infrastructure |
| Zero downtime | Database migrations must be backward-compatible |
| Full testing in production-like environment | Higher infrastructure cost |

## Canary Deployment

Canary deployment routes a small percentage of traffic to the new version. If the canary is healthy, traffic is gradually shifted from the old version to the new version.

```
Canary Deployment Phases:

Phase 1: 5% → canary                Phase 2: 25% → canary
┌────────────────────────┐          ┌────────────────────────┐
│  95% ──▶ v1 (stable)  │          │  75% ──▶ v1 (stable)  │
│   5% ──▶ v2 (canary)  │          │  25% ──▶ v2 (canary)  │
└────────────────────────┘          └────────────────────────┘

Phase 3: 75% → canary               Phase 4: 100% → new stable
┌────────────────────────┐          ┌────────────────────────┐
│  25% ──▶ v1 (stable)  │          │ 100% ──▶ v2 (stable)  │
│  75% ──▶ v2 (canary)  │          │                        │
└────────────────────────┘          └────────────────────────┘
```

### Automated Canary with Flagger

```yaml
# Flagger Canary CRD
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: order-service
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  service:
    port: 8080
  analysis:
    interval: 1m
    threshold: 5          # Max failed checks before rollback
    maxWeight: 50          # Max traffic to canary (50%)
    stepWeight: 10         # Increase canary traffic by 10% per step
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99          # Require 99% success rate
        interval: 1m
      - name: request-duration
        thresholdRange:
          max: 500         # Max P99 latency: 500ms
        interval: 1m
```

## Feature Flags

Feature flags decouple deployment from release. Code is deployed but features are toggled on or off at runtime without redeployment.

```
Deployment:  Code for feature X is deployed to all instances
Release:     Feature X is enabled for 5% of users via flag

  ┌──────────────┐
  │   Service    │
  │              │
  │  if (flag    │
  │    enabled)  │──▶ New feature path
  │  else        │
  │    ...       │──▶ Old feature path
  └──────────────┘
```

### Feature Flag Strategies

| Strategy | Description | Use Case |
|---|---|---|
| **Boolean flag** | On/off toggle | Kill switch for a feature |
| **Percentage rollout** | Enable for N% of requests | Gradual rollout to users |
| **User targeting** | Enable for specific users/groups | Beta testers, internal users |
| **Environment flag** | Enable per environment (staging vs. prod) | Testing in staging before production |

### Feature Flag Tools

| Tool | Type | Key Features |
|---|---|---|
| **LaunchDarkly** | SaaS | Real-time targeting, experimentation, audit logs |
| **Unleash** | Open source | Self-hosted, SDKs for many languages, strategies |
| **Flagsmith** | Open source / SaaS | Feature flags + remote config |
| **AWS AppConfig** | Managed | Feature flags with safe deployment guardrails |

## GitOps

GitOps uses Git as the single source of truth for declarative infrastructure and application configuration. Changes are made via pull requests, and an operator reconciles the desired state in Git with the actual state in the cluster.

### GitOps Workflow

```
Developer                    Git Repository              Kubernetes Cluster
    │                             │                            │
    │  1. Push code change        │                            │
    ├────────────────────────────▶│                            │
    │                             │                            │
    │  2. CI builds + tests       │                            │
    │     + pushes image          │                            │
    │                             │                            │
    │  3. Update manifests        │                            │
    │     (image tag in Git)      │                            │
    ├────────────────────────────▶│                            │
    │                             │  4. GitOps operator        │
    │                             │     detects change         │
    │                             ├───────────────────────────▶│
    │                             │                            │
    │                             │  5. Reconciles cluster     │
    │                             │     to match Git state     │
    │                             │                            │
```

### GitOps Tools

| Tool | Features |
|---|---|
| **ArgoCD** | Kubernetes-native, UI dashboard, multi-cluster, RBAC |
| **Flux** | CNCF project, Git-native, Helm/Kustomize support |
| **Jenkins X** | CI/CD + GitOps, preview environments |

## CI/CD for Microservices

### Pipeline Per Service

Each microservice has its own CI/CD pipeline. A change to one service triggers only that service's pipeline.

```
Monorepo Structure:
  services/
  ├── order-service/        → triggers order-service pipeline
  │   ├── src/
  │   ├── Dockerfile
  │   └── .github/workflows/order-ci.yml
  ├── payment-service/      → triggers payment-service pipeline
  │   ├── src/
  │   ├── Dockerfile
  │   └── .github/workflows/payment-ci.yml
  └── catalog-service/      → triggers catalog-service pipeline
      ├── src/
      ├── Dockerfile
      └── .github/workflows/catalog-ci.yml
```

### Pipeline Stages

```
┌─────────┐    ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌─────────┐
│  Build  │───▶│  Test   │───▶│  Scan    │───▶│  Push    │───▶│ Deploy  │
│         │    │         │    │          │    │  Image   │    │         │
│ Compile │    │ Unit    │    │ SAST     │    │ to       │    │ Staging │
│ Lint    │    │ Contract│    │ Image    │    │ Registry │    │ Canary  │
│         │    │ Integ.  │    │ Vuln scan│    │          │    │ Prod    │
└─────────┘    └─────────┘    └──────────┘    └──────────┘    └─────────┘
```

## Containerization

### Dockerfile Best Practices

```dockerfile
# Multi-stage build for a Go microservice
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /service ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /service /service
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/service"]
```

### Image Best Practices

- ✅ Use multi-stage builds to minimize image size
- ✅ Use distroless or Alpine base images
- ✅ Run as non-root user
- ✅ Pin base image versions (use digest or specific tag)
- ✅ Scan images for vulnerabilities before pushing
- ❌ Do not use `latest` tag in production
- ❌ Do not install unnecessary packages

## Choosing a Strategy

| Strategy | Risk | Rollback Speed | Infrastructure Cost | Best For |
|---|---|---|---|---|
| **Rolling update** | Medium | Slow (re-rollout) | Low | Standard releases |
| **Blue-green** | Low | Instant (switch) | High (2x infra) | Critical services |
| **Canary** | Low | Fast (route to stable) | Medium | Data-driven validation |
| **Feature flags** | Very Low | Instant (toggle off) | Low | Progressive delivery |

## Best Practices

- ✅ Deploy each microservice independently — no coordinated multi-service releases
- ✅ Use immutable container images — same image from staging to production
- ✅ Automate everything — builds, tests, scans, deployments
- ✅ Implement GitOps for declarative, auditable deployments
- ✅ Use canary or blue-green for high-risk changes
- ✅ Always have a rollback plan and test it regularly
- ❌ Do not deploy multiple services in a single deployment artifact
- ❌ Do not skip staging environments
- ❌ Do not deploy on Fridays (or before holidays) without automated rollback

## Next Steps

Continue to [Testing Strategies](10-TESTING-STRATEGIES.md) to learn about the testing pyramid for microservices, contract testing, integration testing, end-to-end testing, and chaos engineering.
