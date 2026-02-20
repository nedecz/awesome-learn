# Continuous Delivery & Deployment

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Continuous Delivery vs Continuous Deployment](#continuous-delivery-vs-continuous-deployment)
   - [Key Difference: Manual Gate vs Automatic](#key-difference-manual-gate-vs-automatic)
   - [Maturity Model](#maturity-model)
   - [Choosing the Right Approach](#choosing-the-right-approach)
3. [Deployment Pipelines](#deployment-pipelines)
   - [Multi-Stage Pipelines](#multi-stage-pipelines)
   - [Promotion Between Environments](#promotion-between-environments)
   - [Pipeline as Code](#pipeline-as-code)
4. [Environment Management](#environment-management)
   - [Standard Environment Tiers](#standard-environment-tiers)
   - [Environment Parity](#environment-parity)
   - [Environment Provisioning](#environment-provisioning)
   - [Ephemeral Environments](#ephemeral-environments)
5. [Release Strategies](#release-strategies)
   - [Blue-Green Deployments](#blue-green-deployments)
   - [Canary Releases](#canary-releases)
   - [Rolling Updates](#rolling-updates)
   - [Feature Flags](#feature-flags)
   - [A/B Testing](#ab-testing)
   - [Shadow Traffic](#shadow-traffic)
   - [Strategy Comparison](#strategy-comparison)
6. [Rollback Strategies](#rollback-strategies)
   - [Automatic Rollback](#automatic-rollback)
   - [Version Pinning](#version-pinning)
   - [Database Rollback Considerations](#database-rollback-considerations)
7. [Deployment Automation](#deployment-automation)
   - [Infrastructure Provisioning](#infrastructure-provisioning)
   - [Configuration Management](#configuration-management)
   - [GitOps](#gitops)
8. [Approval Gates and Manual Interventions](#approval-gates-and-manual-interventions)
   - [Gate Types](#gate-types)
   - [Approval Workflows](#approval-workflows)
   - [Compliance and Audit Trails](#compliance-and-audit-trails)
9. [Database Migrations in CD Pipelines](#database-migrations-in-cd-pipelines)
   - [Migration Strategies](#migration-strategies)
   - [Backward-Compatible Migrations](#backward-compatible-migrations)
   - [Migration Pipeline Integration](#migration-pipeline-integration)
10. [Monitoring and Verification After Deployment](#monitoring-and-verification-after-deployment)
    - [Smoke Tests](#smoke-tests)
    - [Synthetic Monitoring](#synthetic-monitoring)
    - [Deployment Verification Pipeline](#deployment-verification-pipeline)
11. [Release Orchestration](#release-orchestration)
    - [Release Trains](#release-trains)
    - [Semantic Versioning](#semantic-versioning)
    - [Changelogs](#changelogs)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **Continuous Delivery (CD)** and **Continuous Deployment** — the practices of automatically and reliably releasing software changes through a series of environments, from initial commit all the way to production. It covers deployment pipelines, environment management, release strategies, rollback techniques, database migrations, and post-deployment verification for teams operating at any scale.

### Target Audience

- **DevOps Engineers** designing and maintaining deployment pipelines and release processes
- **Platform Engineers** building internal deployment platforms and environment management tooling
- **SREs** ensuring reliable, zero-downtime deployments and rapid rollback capabilities
- **Engineering Managers** establishing release cadences, approval workflows, and deployment metrics

### Scope

- Distinguishing Continuous Delivery from Continuous Deployment
- Multi-stage deployment pipelines and promotion workflows
- Environment tiers, parity, and ephemeral environments
- Release strategies: blue-green, canary, rolling, feature flags, A/B testing, and shadow traffic
- Rollback strategies and database migration safety
- Deployment automation with infrastructure as code and GitOps
- Approval gates, compliance, and audit trails
- Post-deployment verification: smoke tests and synthetic monitoring
- Release orchestration, semantic versioning, and changelogs

---

## Continuous Delivery vs Continuous Deployment

Both Continuous Delivery and Continuous Deployment extend Continuous Integration by automating the path from a tested build to a running service. The critical distinction is whether a **human decision** sits between the final test stage and the production release.

### Key Difference: Manual Gate vs Automatic

```
Continuous Delivery                        Continuous Deployment
┌───────────┐                              ┌───────────┐
│   Commit   │                              │   Commit   │
└─────┬─────┘                              └─────┬─────┘
      ▼                                          ▼
┌───────────┐                              ┌───────────┐
│   Build    │                              │   Build    │
└─────┬─────┘                              └─────┬─────┘
      ▼                                          ▼
┌───────────┐                              ┌───────────┐
│   Test     │                              │   Test     │
└─────┬─────┘                              └─────┬─────┘
      ▼                                          ▼
┌───────────┐                              ┌───────────┐
│  Staging   │                              │  Staging   │
└─────┬─────┘                              └─────┬─────┘
      ▼                                          ▼
┌─────────────┐                            ┌───────────┐
│ Manual Gate  │ ◄── Human approves        │ Production │ ◄── Automatic
│  (approve)   │     or rejects            │  (live)    │     on pass
└─────┬───────┘                            └───────────┘
      ▼
┌───────────┐
│ Production │
└───────────┘
```

| Aspect | Continuous Delivery | Continuous Deployment |
|---|---|---|
| **Production release** | Manual approval required | Fully automated |
| **Human gate** | Yes — explicit approve/reject step | No — pipeline runs end-to-end |
| **Release frequency** | On-demand (hours to days) | Every passing commit (minutes) |
| **Risk tolerance** | Lower — human reviews before release | Higher — requires robust automated checks |
| **Test requirements** | High | Very high — tests are the only safety net |
| **Rollback model** | Manual or semi-automated | Fully automated |
| **Common in** | Regulated industries, enterprise | SaaS, web applications, startups |
| **Pipeline complexity** | Moderate | High — must handle all edge cases |

### Maturity Model

Organizations typically progress through stages of deployment maturity:

```
Level 0              Level 1              Level 2              Level 3
Manual               CI Only              Continuous           Continuous
Deployment                                Delivery             Deployment
┌──────────┐        ┌──────────┐        ┌──────────┐        ┌──────────┐
│ FTP /     │   →    │ Automated │   →    │ Automated │   →    │ Automated │
│ manual    │        │ build     │        │ pipeline  │        │ pipeline  │
│ copy to   │        │ and test  │        │ to staging│        │ to prod   │
│ server    │        │ only      │        │ + manual  │        │ no human  │
│           │        │           │        │ prod gate │        │ gate      │
└──────────┘        └──────────┘        └──────────┘        └──────────┘
  Weeks               Days                Hours                Minutes
```

### Choosing the Right Approach

**Choose Continuous Delivery when:**

- Regulatory requirements mandate manual approval (SOC 2, HIPAA, PCI-DSS)
- Deployments involve coordinated releases across multiple teams
- Business stakeholders need to control release timing (marketing launches, feature announcements)
- The team is building deployment confidence and maturing its test suite

**Choose Continuous Deployment when:**

- The team has high confidence in automated tests (unit, integration, end-to-end, contract)
- Fast iteration speed is a competitive advantage
- The application supports feature flags for decoupling deploy from release
- Robust automated rollback mechanisms are in place

---

## Deployment Pipelines

A **deployment pipeline** is the automated process that takes code from version control through build, test, and release stages until it reaches production. Each stage acts as a quality gate — if any stage fails, the pipeline stops and the team is notified.

### Multi-Stage Pipelines

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Source   │──▶│  Build   │──▶│  Test    │──▶│  Stage   │──▶│  Prod    │
│          │   │          │   │          │   │          │   │          │
│ • Clone  │   │ • Compile│   │ • Unit   │   │ • Deploy │   │ • Deploy │
│ • Lint   │   │ • Package│   │ • Integ  │   │ • Smoke  │   │ • Smoke  │
│ • Secrets│   │ • Image  │   │ • E2E    │   │ • Perf   │   │ • Monitor│
│   scan   │   │   build  │   │ • SAST   │   │ • UAT    │   │ • Alert  │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
     5 min          5 min         10 min         15 min          10 min
                                                          Total: ~45 min
```

Each stage should be:

- **Idempotent** — running the same stage twice produces the same result
- **Isolated** — stages do not share mutable state; artifacts pass between them explicitly
- **Fast-failing** — cheaper and faster checks run first to provide rapid feedback
- **Observable** — every stage emits logs, metrics, and status updates

### Promotion Between Environments

Artifact promotion ensures that the **exact same artifact** tested in one environment is deployed to the next. Never rebuild between environments — rebuild introduces variance.

```
                          Artifact Promotion Flow

    ┌─────────────────────────────────────────────────────────────┐
    │                   Artifact Repository                       │
    │                                                             │
    │   app:1.2.3-rc.1  ──▶  app:1.2.3-rc.1  ──▶  app:1.2.3    │
    │   (tagged for dev)     (promoted to stg)    (promoted to   │
    │                                               prod)         │
    └─────────────────────────────────────────────────────────────┘
         │                       │                      │
         ▼                       ▼                      ▼
    ┌──────────┐           ┌──────────┐          ┌──────────┐
    │   Dev    │           │ Staging  │          │Production│
    │          │           │          │          │          │
    │ Smoke    │  Pass ──▶ │ Integ    │ Pass ──▶ │ Canary   │
    │ tests    │           │ Perf     │          │ Full     │
    └──────────┘           └──────────┘          └──────────┘
```

**Key principles:**

- **Build once, deploy many** — the same container image or binary is promoted, not rebuilt
- **Environment-specific configuration** — injected at deploy time via environment variables, secrets managers, or config maps
- **Immutable artifacts** — artifacts are versioned and never overwritten

### Pipeline as Code

Define pipelines in version-controlled files alongside your application code so that pipeline changes go through the same review process as code changes.

```yaml
# .github/workflows/deploy.yml — GitHub Actions multi-stage pipeline
name: Deploy Pipeline

on:
  push:
    branches: [main]

env:
  IMAGE: ghcr.io/myorg/myapp
  VERSION: ${{ github.sha }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Build and push image
        id: meta
        run: |
          docker build -t $IMAGE:$VERSION .
          docker push $IMAGE:$VERSION
          echo "tags=$IMAGE:$VERSION" >> "$GITHUB_OUTPUT"

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    steps:
      - name: Deploy to staging
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ needs.build.outputs.image-tag }} \
            --namespace=staging

      - name: Run smoke tests
        run: |
          curl --fail --retry 5 --retry-delay 10 \
            https://staging.myapp.com/healthz

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    steps:
      - name: Deploy canary (10%)
        run: |
          kubectl set image deployment/myapp-canary \
            myapp=${{ needs.build.outputs.image-tag }} \
            --namespace=production

      - name: Verify canary metrics
        run: |
          sleep 300
          ./scripts/verify-canary-metrics.sh

      - name: Promote to full production
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ needs.build.outputs.image-tag }} \
            --namespace=production
```

---

## Environment Management

Environments are isolated instances of your application stack where different stages of testing and validation occur. Effective environment management is foundational to reliable Continuous Delivery.

### Standard Environment Tiers

```
┌────────────────────────────────────────────────────────────────────────┐
│                      Environment Progression                           │
│                                                                        │
│  ┌─────────┐    ┌─────────┐    ┌──────────┐    ┌──────────────────┐  │
│  │   Dev   │───▶│   QA    │───▶│ Pre-Prod │───▶│   Production     │  │
│  │         │    │         │    │ (Staging) │    │                  │  │
│  │ Develop │    │ Test    │    │ Validate  │    │  Serve traffic   │  │
│  │ & debug │    │ features│    │ release   │    │  real users      │  │
│  └─────────┘    └─────────┘    └──────────┘    └──────────────────┘  │
│                                                                        │
│  Low fidelity ──────────────────────────────── High fidelity          │
│  Fast feedback ─────────────────────────────── Slow feedback          │
│  Low cost ──────────────────────────────────── High cost              │
└────────────────────────────────────────────────────────────────────────┘
```

| Environment | Purpose | Data | Scale | Access |
|---|---|---|---|---|
| **Development** | Feature development, local testing | Synthetic / seed data | Minimal (1 replica) | Developers |
| **QA / Test** | Functional and integration testing | Anonymized subset of prod | Small (1–2 replicas) | QA team, developers |
| **Pre-Production (Staging)** | Release validation, performance testing | Production-like (anonymized) | Production-scale | Engineering, product |
| **Production** | Serve real user traffic | Real data | Full scale (auto-scaled) | Operations, SRE |

### Environment Parity

Environment drift — differences between staging and production — is one of the most common causes of deployment failures. Strive for maximum parity across environments.

**Sources of environment drift:**

| Drift Category | Example | Mitigation |
|---|---|---|
| **Infrastructure** | Staging uses smaller instances than prod | Use IaC with parameterized sizing |
| **Configuration** | Different feature flags or timeout values | Centralized config management |
| **Data** | Staging has stale or empty databases | Automated data seeding from anonymized prod |
| **Dependencies** | Different versions of databases or caches | Pin dependency versions in IaC |
| **Networking** | Different DNS, load balancer, or firewall rules | Replicate network topology with IaC |
| **Secrets** | Missing or outdated credentials in staging | Secrets rotation synced across environments |

```yaml
# terraform/environments/staging/main.tf — parameterized environment config
# Uses the same module as production with different parameters
module "app" {
  source = "../../modules/app"

  environment    = "staging"
  instance_type  = "t3.medium"       # prod uses t3.xlarge
  min_replicas   = 2                  # prod uses 5
  max_replicas   = 4                  # prod uses 20
  db_instance    = "db.r6g.large"    # prod uses db.r6g.2xlarge

  # Same application image as production
  app_image      = var.app_image
  app_version    = var.app_version
}
```

### Environment Provisioning

Automate environment creation so any team member can spin up a full environment on demand.

```yaml
# docker-compose.env.yml — local environment matching production topology
version: "3.9"

services:
  app:
    image: myapp:${VERSION:-latest}
    environment:
      - DATABASE_URL=postgres://app:secret@db:5432/myapp
      - REDIS_URL=redis://cache:6379
      - LOG_LEVEL=debug
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d myapp"]
      interval: 5s
      timeout: 3s
      retries: 5

  cache:
    image: redis:7-alpine

  worker:
    image: myapp:${VERSION:-latest}
    command: ["./worker"]
    environment:
      - DATABASE_URL=postgres://app:secret@db:5432/myapp
      - REDIS_URL=redis://cache:6379
```

### Ephemeral Environments

Ephemeral (short-lived) environments are created automatically for each pull request and destroyed when the PR is merged or closed. They provide isolated testing without environment contention.

```yaml
# .github/workflows/preview.yml — ephemeral environment per PR
name: Preview Environment

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  deploy-preview:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Create preview namespace
        run: |
          kubectl create namespace pr-${{ github.event.number }} \
            --dry-run=client -o yaml | kubectl apply -f -

      - name: Deploy preview
        run: |
          helm upgrade --install myapp-pr-${{ github.event.number }} \
            ./charts/myapp \
            --namespace pr-${{ github.event.number }} \
            --set image.tag=${{ github.sha }} \
            --set ingress.host=pr-${{ github.event.number }}.preview.myapp.com

      - name: Comment PR with preview URL
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '🚀 Preview deployed: https://pr-${{ github.event.number }}.preview.myapp.com'
            })

  teardown-preview:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Delete preview namespace
        run: |
          kubectl delete namespace pr-${{ github.event.number }} --ignore-not-found
```

---

## Release Strategies

Release strategies determine **how** new versions of your application are exposed to users. The right strategy balances deployment speed, risk, and operational complexity.

### Blue-Green Deployments

Blue-green deployment maintains two identical production environments. At any time, one (blue) serves live traffic while the other (green) receives the new version. Traffic is switched atomically by updating the load balancer or DNS.

```
                        Blue-Green Deployment

    Step 1: Blue is live                Step 2: Deploy to Green
    ┌──────────────┐                    ┌──────────────┐
    │ Load Balancer │                    │ Load Balancer │
    └──────┬───────┘                    └──────┬───────┘
           │ 100%                              │ 100%
           ▼                                   ▼
    ┌──────────────┐                    ┌──────────────┐
    │  Blue (v1)   │ ◄── Live          │  Blue (v1)   │ ◄── Live
    │  Running     │                    │  Running     │
    └──────────────┘                    └──────────────┘
    ┌──────────────┐                    ┌──────────────┐
    │  Green       │                    │  Green (v2)  │ ◄── Deploy
    │  Idle        │                    │  Testing     │     & test
    └──────────────┘                    └──────────────┘

    Step 3: Switch traffic              Step 4: Green is live
    ┌──────────────┐                    ┌──────────────┐
    │ Load Balancer │                    │ Load Balancer │
    └──────┬───────┘                    └──────┬───────┘
           │ 100%                              │ 100%
           ▼                                   ▼
    ┌──────────────┐                    ┌──────────────┐
    │  Blue (v1)   │                    │  Blue (v1)   │ ◄── Standby
    │  Standby     │                    │  (rollback)  │
    └──────────────┘                    └──────────────┘
    ┌──────────────┐                    ┌──────────────┐
    │  Green (v2)  │ ◄── Live          │  Green (v2)  │ ◄── Live
    │  Serving     │                    │  Serving     │
    └──────────────┘                    └──────────────┘
```

**Advantages:**

- Instant rollback — switch traffic back to the previous environment
- Zero downtime — traffic cutover is atomic
- Full environment testing before serving real traffic

**Disadvantages:**

- Requires double the infrastructure (cost)
- Database schema changes need careful coordination
- Long-running connections may be disrupted during switchover

### Canary Releases

A canary release gradually shifts a small percentage of traffic to the new version while monitoring for errors. If metrics stay healthy, traffic is progressively increased until the new version serves 100%.

```
                          Canary Release Progression

    Phase 1: 5% canary         Phase 2: 25% canary        Phase 3: 100% rollout
    ┌──────────────┐           ┌──────────────┐           ┌──────────────┐
    │ Load Balancer │           │ Load Balancer │           │ Load Balancer │
    └──────┬───────┘           └──────┬───────┘           └──────┬───────┘
           │                          │                          │
      ┌────┴────┐                ┌────┴────┐                     │
      │         │                │         │                     │
   95%│      5% │             75%│      25%│                100% │
      ▼         ▼                ▼         ▼                     ▼
  ┌───────┐ ┌───────┐       ┌───────┐ ┌───────┐           ┌───────┐
  │  v1   │ │  v2   │       │  v1   │ │  v2   │           │  v2   │
  │(stable)│ │(canary)│       │(stable)│ │(canary)│           │(stable)│
  └───────┘ └───────┘       └───────┘ └───────┘           └───────┘

   Monitor error rate,         If healthy,                  Old version
   latency, saturation         increase traffic             decommissioned
```

```yaml
# kubernetes canary with Argo Rollouts
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: success-rate
            args:
              - name: service-name
                value: myapp
        - setWeight: 25
        - pause: { duration: 10m }
        - analysis:
            templates:
              - templateName: success-rate
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
      canaryService: myapp-canary
      stableService: myapp-stable
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
          image: myapp:2.0.0
          ports:
            - containerPort: 8080
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 60s
      successCondition: result[0] >= 0.99
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",
              status=~"2.."}[5m])) /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
```

### Rolling Updates

A rolling update incrementally replaces old instances with new ones. At no point is the entire fleet running the old or new version — both coexist during the rollout.

```
                         Rolling Update (4 replicas)

    Start          Step 1         Step 2         Step 3         Done
  ┌──┐┌──┐┌──┐┌──┐  ┌──┐┌──┐┌──┐┌──┐  ┌──┐┌──┐┌──┐┌──┐  ┌──┐┌──┐┌──┐┌──┐  ┌──┐┌──┐┌──┐┌──┐
  │v1││v1││v1││v1│  │v2││v1││v1││v1│  │v2││v2││v1││v1│  │v2││v2││v2││v1│  │v2││v2││v2││v2│
  └──┘└──┘└──┘└──┘  └──┘└──┘└──┘└──┘  └──┘└──┘└──┘└──┘  └──┘└──┘└──┘└──┘  └──┘└──┘└──┘└──┘

  100% v1           75% v1 / 25% v2   50% / 50%          25% v1 / 75% v2   100% v2
```

```yaml
# Kubernetes rolling update configuration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # at most 1 pod down during update
      maxSurge: 1           # at most 1 extra pod during update
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
          image: myapp:2.0.0
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
```

### Feature Flags

Feature flags decouple **deployment** from **release**. Code is deployed to production with new features hidden behind flags, which can be toggled on or off without redeployment.

```
                    Feature Flag Lifecycle

    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │  Create  │───▶│  Test    │───▶│  Release │───▶│  Remove  │
    │  Flag    │    │  Flag    │    │  Flag    │    │  Flag    │
    │          │    │          │    │          │    │          │
    │ Add flag │    │ Enable   │    │ Gradual  │    │ Remove   │
    │ in code  │    │ for devs │    │ rollout  │    │ flag and │
    │ default  │    │ and QA   │    │ to all   │    │ dead code│
    │ = off    │    │ only     │    │ users    │    │          │
    └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

```yaml
# feature flag configuration example (LaunchDarkly-style)
flags:
  new-checkout-flow:
    description: "Redesigned checkout experience"
    type: boolean
    default: false
    environments:
      development:
        enabled: true
        targets:
          - variation: true
            values: ["dev-team@company.com"]
      staging:
        enabled: true
        rules:
          - variation: true
            percentage: 100
      production:
        enabled: true
        rules:
          - variation: true
            percentage: 10       # 10% of users see new checkout
        fallthrough:
          variation: false        # 90% see old checkout
```

### A/B Testing

A/B testing is a data-driven release strategy where users are split between variants to measure the impact on business metrics. Unlike canary releases (which focus on technical health), A/B tests measure **user behavior** — conversion rates, engagement, revenue.

```
                          A/B Test Architecture

                    ┌──────────────────┐
                    │   Traffic Router  │
                    │  (splits by user  │
                    │   hash / cohort)  │
                    └────────┬─────────┘
                         ┌───┴───┐
                    50%  │       │  50%
                         ▼       ▼
                   ┌──────┐   ┌──────┐
                   │  A   │   │  B   │
                   │(ctrl)│   │(test)│
                   └──┬───┘   └──┬───┘
                      │          │
                      ▼          ▼
               ┌──────────────────────┐
               │   Analytics / Metrics │
               │                      │
               │  Conversion: A=2.1%  │
               │               B=2.8% │
               │  p-value: 0.003      │
               │  Winner: B ✓         │
               └──────────────────────┘
```

### Shadow Traffic

Shadow (or dark) traffic deployment sends a copy of production requests to the new version without returning its responses to users. This allows testing with real traffic patterns without any user impact.

```
                        Shadow Traffic Pattern

    ┌────────┐      ┌──────────────────┐
    │  User  │─────▶│   Load Balancer   │
    └────────┘      └────────┬─────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
              Response to user   Copy of request
                    │            (fire-and-forget)
                    ▼                 ▼
              ┌──────────┐     ┌──────────┐
              │  v1      │     │  v2      │
              │ (live)   │     │ (shadow) │
              │          │     │          │
              │ Response │     │ Response │
              │ returned │     │ discarded│
              └──────────┘     └──────────┘
                                    │
                                    ▼
                            ┌──────────────┐
                            │  Compare     │
                            │  responses,  │
                            │  latency,    │
                            │  errors      │
                            └──────────────┘
```

**Use cases:**

- Validating new versions under real traffic load without user risk
- Comparing response correctness between old and new versions
- Load testing with production traffic patterns

**Caution:** Shadow traffic must not trigger side effects — writes, emails, charges, or external API calls must be suppressed in the shadow instance.

### Strategy Comparison

| Strategy | Downtime | Rollback Speed | Resource Cost | Complexity | User Impact on Failure |
|---|---|---|---|---|---|
| **Blue-Green** | None | Instant (seconds) | 2× infrastructure | Low | None (instant switch) |
| **Canary** | None | Fast (minutes) | 1× + small canary | Medium | Small % of users |
| **Rolling** | None | Moderate (minutes) | 1× + surge capacity | Low | Partial (during rollout) |
| **Feature Flags** | None | Instant (flag off) | 1× | Medium | Controlled by flag scope |
| **A/B Testing** | None | Instant (flag off) | 1× + variant infra | High | 50% of test population |
| **Shadow** | None | N/A (no user traffic) | 2× traffic processing | High | None |

---

## Rollback Strategies

No matter how thorough your testing, some deployments will fail in production. A robust rollback strategy is essential to minimize user impact.

### Automatic Rollback

Configure your deployment system to automatically roll back when health checks fail or error rates exceed thresholds.

```yaml
# Kubernetes automatic rollback on failed health checks
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  minReadySeconds: 30
  progressDeadlineSeconds: 300   # rollback if not complete in 5 minutes
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:2.0.0
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
```

```yaml
# Argo Rollouts automatic rollback on metric degradation
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: error-rate-check
      abortScaleDownDelaySeconds: 30   # time before scaling down aborted
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:2.0.0
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
    - name: error-rate
      interval: 30s
      failureLimit: 3
      successCondition: result[0] < 0.05
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{status=~"5.."}[2m])) /
            sum(rate(http_requests_total[2m]))
```

### Version Pinning

Always pin exact versions for production deployments so rollback is a matter of redeploying a known-good version.

```yaml
# deployment manifest with pinned image digest (not just tag)
spec:
  containers:
    - name: myapp
      # Tags can be overwritten — digests are immutable
      image: myapp@sha256:a1b2c3d4e5f6...
```

```bash
# Quick rollback to previous version in Kubernetes
kubectl rollout undo deployment/myapp --namespace=production

# Rollback to a specific revision
kubectl rollout undo deployment/myapp --to-revision=42 --namespace=production

# Check rollout history
kubectl rollout history deployment/myapp --namespace=production
```

### Database Rollback Considerations

Application rollbacks are straightforward when stateless — restart with the old image. Database changes are far more complex because data transformations may not be reversible.

```
                   Database Rollback Complexity

    ┌─────────────┐    Easy         ┌─────────────┐    Hard
    │ Add column  │   rollback      │ Drop column │   rollback
    │ (nullable)  │ ◄──────────     │             │ ◄──────────
    │             │   Just drop     │             │   Data is
    │             │   the column    │             │   gone!
    └─────────────┘                 └─────────────┘

    ┌─────────────┐    Medium       ┌─────────────┐    Very Hard
    │ Rename      │   rollback      │ Transform   │   rollback
    │ column      │ ◄──────────     │ data values │ ◄──────────
    │             │   Rename back   │             │   Original
    │             │                 │             │   values lost
    └─────────────┘                 └─────────────┘
```

**Best practices for database rollback safety:**

- Always write **forward-only migrations** that are backward-compatible
- Use the expand-and-contract pattern (add new → migrate data → remove old)
- Never drop columns or tables in the same release that deploys new code
- Maintain rollback scripts and test them in staging
- Back up the database before running irreversible migrations

---

## Deployment Automation

Deployment automation eliminates manual steps, reducing human error and enabling repeatable, auditable releases.

### Infrastructure Provisioning

Use Infrastructure as Code (IaC) to define and version all infrastructure alongside application code.

```yaml
# terraform/main.tf — production infrastructure
resource "aws_ecs_service" "app" {
  name            = "myapp"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.app_replicas
  launch_type     = "FARGATE"

  deployment_controller {
    type = "CODE_DEPLOY"    # enables blue-green via CodeDeploy
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "myapp"
    container_port   = 8080
  }

  network_configuration {
    subnets         = var.private_subnets
    security_groups = [aws_security_group.app.id]
  }
}

resource "aws_ecs_task_definition" "app" {
  family                   = "myapp"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.cpu
  memory                   = var.memory
  network_mode             = "awsvpc"

  container_definitions = jsonencode([{
    name  = "myapp"
    image = "${var.ecr_repo}:${var.app_version}"
    portMappings = [{
      containerPort = 8080
      protocol      = "tcp"
    }]
    environment = [
      { name = "ENV", value = var.environment }
    ]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"  = "/ecs/myapp"
        "awslogs-region" = var.region
      }
    }
  }])
}
```

### Configuration Management

Separate configuration from code. Configuration should be injected at deploy time and vary per environment.

| Method | Tool Examples | Best For |
|---|---|---|
| **Environment variables** | Docker env, K8s ConfigMaps | Simple key-value settings |
| **Secrets managers** | Vault, AWS Secrets Manager, Azure Key Vault | Credentials, API keys, certificates |
| **Config files** | Consul, etcd, Spring Cloud Config | Structured configuration |
| **Feature flag services** | LaunchDarkly, Unleash, Flagsmith | Runtime feature toggles |

```yaml
# Kubernetes ConfigMap and Secret injection
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  CACHE_TTL: "300"
---
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgres://user:pass@db:5432/myapp"
  API_KEY: "sk-xxxx"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:2.0.0
          envFrom:
            - configMapRef:
                name: myapp-config
            - secretRef:
                name: myapp-secrets
```

### GitOps

GitOps uses Git as the single source of truth for both application code and infrastructure configuration. A GitOps operator continuously reconciles the cluster state with the desired state declared in Git.

```
                         GitOps Workflow

    ┌──────────┐     ┌───────────┐     ┌──────────────────┐
    │Developer │────▶│  Git Repo  │────▶│  GitOps Operator  │
    │          │     │            │     │  (Argo CD / Flux) │
    │ Commits  │     │ Desired    │     │                  │
    │ manifest │     │ state      │     │  Watches repo    │
    │ changes  │     │ stored     │     │  Detects drift   │
    └──────────┘     └───────────┘     │  Reconciles      │
                                        └────────┬─────────┘
                                                 │
                                                 ▼
                                        ┌──────────────────┐
                                        │  Kubernetes      │
                                        │  Cluster         │
                                        │                  │
                                        │  Actual state    │
                                        │  matches desired │
                                        └──────────────────┘
```

```yaml
# Argo CD Application manifest
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/k8s-manifests.git
    targetRevision: main
    path: environments/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true           # delete resources removed from Git
      selfHeal: true         # revert manual cluster changes
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 1m
```

---

## Approval Gates and Manual Interventions

Approval gates introduce controlled checkpoints in the deployment pipeline where human review is required before proceeding. They balance deployment speed with risk management.

### Gate Types

| Gate Type | Trigger | Purpose | Example |
|---|---|---|---|
| **Manual approval** | Human clicks approve | Risk mitigation, compliance | Production deployment sign-off |
| **Scheduled window** | Time-based | Deploy only during low-traffic hours | Weekday 10am–4pm only |
| **Metric gate** | Automated metric check | Verify health before promotion | Error rate < 1% for 10 min |
| **External gate** | Third-party system check | Security, compliance validation | Security scan passed |
| **Change advisory board** | Committee review | High-risk change governance | Infrastructure changes |

### Approval Workflows

```yaml
# GitHub Actions environment protection rules
# (configured in repository Settings → Environments)

# Pipeline referencing protected environments
name: Production Deploy

on:
  workflow_dispatch:

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging          # no approval required
    steps:
      - name: Deploy to staging
        run: ./deploy.sh staging

  approval:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production       # requires approval from designated reviewers
    steps:
      - name: Deploy to production
        run: ./deploy.sh production
```

```yaml
# Azure DevOps approval gate
stages:
  - stage: DeployProduction
    displayName: "Deploy to Production"
    dependsOn: DeployStaging
    condition: succeeded()
    jobs:
      - deployment: ProductionDeploy
        environment: "production"   # environment has approval checks configured
        strategy:
          runOnce:
            deploy:
              steps:
                - script: ./deploy.sh production
```

### Compliance and Audit Trails

For regulated environments, every deployment must produce an audit trail documenting:

- **Who** approved the deployment
- **What** was deployed (artifact version, commit SHA)
- **When** the deployment occurred
- **Where** it was deployed (environment, region)
- **Why** — linked ticket or change request

```
                    Audit Trail Record

    ┌──────────────────────────────────────────────────┐
    │ Deployment #4521                                  │
    │                                                  │
    │ Artifact:    myapp:2.1.0@sha256:a1b2c3...        │
    │ Commit:      abc123def456                        │
    │ Environment: production (us-east-1)              │
    │ Requested:   2025-03-15T14:30:00Z                │
    │ Approved by: jane.smith@company.com              │
    │ Approved at: 2025-03-15T14:35:00Z                │
    │ Deployed at: 2025-03-15T14:36:12Z                │
    │ Ticket:      JIRA-4521                           │
    │ Status:      SUCCESS                             │
    │ Rollback:    Available (previous: myapp:2.0.3)   │
    └──────────────────────────────────────────────────┘
```

---

## Database Migrations in CD Pipelines

Database schema changes are among the most challenging aspects of Continuous Delivery because they involve persistent state that survives application restarts and cannot be easily rolled back.

### Migration Strategies

| Strategy | Description | Risk | Speed |
|---|---|---|---|
| **Version-controlled migrations** | Sequential numbered scripts (Flyway, Liquibase, Alembic) | Low | Moderate |
| **State-based migrations** | Compare desired schema to actual, generate diff | Medium | Fast |
| **ORM auto-migration** | Framework generates DDL from model changes | High | Fast |

### Backward-Compatible Migrations

The expand-and-contract pattern ensures zero-downtime database changes by separating the migration into phases that are each backward-compatible.

```
          Expand and Contract Pattern — Rename Column Example

    Phase 1: Expand                Phase 2: Migrate             Phase 3: Contract
    (add new column)               (backfill + dual-write)      (remove old column)

    ┌──────────────────┐          ┌──────────────────┐          ┌──────────────────┐
    │ users table      │          │ users table      │          │ users table      │
    │                  │          │                  │          │                  │
    │ name  (existing) │          │ name  (old)      │          │ full_name (new)  │
    │ full_name (new)  │          │ full_name (new)  │          │                  │
    │                  │          │                  │          │ ✓ Old column     │
    │ App v1 reads:    │          │ App v2 reads:    │          │   dropped after  │
    │   name           │          │   full_name      │          │   v1 is fully    │
    │ App v1 writes:   │          │ App v2 writes:   │          │   decommissioned │
    │   name + full_name│          │   full_name      │          │                  │
    └──────────────────┘          └──────────────────┘          └──────────────────┘

    Deploy migration + app v1     Deploy app v2                 Deploy migration
    simultaneously                after data backfill           after v1 is gone
```

```sql
-- Phase 1: Expand — add new column (backward-compatible)
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);

-- Backfill existing data
UPDATE users SET full_name = name WHERE full_name IS NULL;

-- Phase 2: Application code updated to read/write full_name

-- Phase 3: Contract — remove old column (only after all app instances use full_name)
ALTER TABLE users DROP COLUMN name;
```

### Migration Pipeline Integration

Run migrations as a dedicated pipeline step **before** deploying the new application version. This ensures the schema is ready when the new code starts.

```yaml
# GitHub Actions — migration before deployment
jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run database migrations
        run: |
          flyway -url=${{ secrets.DATABASE_URL }} \
                 -user=${{ secrets.DB_USER }} \
                 -password=${{ secrets.DB_PASSWORD }} \
                 migrate

      - name: Verify migration
        run: |
          flyway -url=${{ secrets.DATABASE_URL }} \
                 -user=${{ secrets.DB_USER }} \
                 -password=${{ secrets.DB_PASSWORD }} \
                 info

  deploy:
    needs: migrate
    runs-on: ubuntu-latest
    steps:
      - name: Deploy application
        run: ./deploy.sh production
```

**Migration safety checklist:**

- [ ] Migration is backward-compatible with the currently running application version
- [ ] Migration has been tested against a copy of production data
- [ ] Migration includes a rollback script (where possible)
- [ ] Long-running migrations use batched operations to avoid table locks
- [ ] Production database has been backed up before migration

---

## Monitoring and Verification After Deployment

Deploying code is only half the battle. Post-deployment verification confirms that the release is healthy and serving users correctly.

### Smoke Tests

Smoke tests are a minimal set of critical-path tests that run immediately after deployment to verify the application is functional.

```yaml
# smoke-tests.yml — post-deployment verification
smoke_tests:
  - name: Health endpoint responds
    request:
      method: GET
      url: "https://${TARGET_HOST}/healthz"
    expect:
      status: 200
      body:
        contains: "ok"

  - name: API authentication works
    request:
      method: POST
      url: "https://${TARGET_HOST}/api/v1/auth/token"
      headers:
        Content-Type: application/json
      body: |
        {"client_id": "${SMOKE_CLIENT_ID}", "client_secret": "${SMOKE_CLIENT_SECRET}"}
    expect:
      status: 200
      body:
        json_path: "$.access_token"
        not_empty: true

  - name: Database connectivity
    request:
      method: GET
      url: "https://${TARGET_HOST}/api/v1/status/db"
    expect:
      status: 200
      body:
        json_path: "$.connected"
        equals: true

  - name: Critical user flow — list items
    request:
      method: GET
      url: "https://${TARGET_HOST}/api/v1/items?limit=1"
      headers:
        Authorization: "Bearer ${SMOKE_TOKEN}"
    expect:
      status: 200
      max_latency_ms: 500
```

### Synthetic Monitoring

Synthetic monitoring continuously runs automated transactions against production to detect issues before users report them.

```
              Synthetic Monitoring Architecture

    ┌──────────────────────────────────────────────────────┐
    │                Synthetic Monitor                      │
    │         (runs every 1–5 minutes)                     │
    │                                                      │
    │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐    │
    │  │ Login  │  │ Search │  │ Add to │  │Checkout│    │
    │  │  flow  │─▶│  item  │─▶│  cart  │─▶│  flow  │    │
    │  └────────┘  └────────┘  └────────┘  └────────┘    │
    │                                                      │
    │  Measures: latency, errors, availability             │
    └──────────────────────┬───────────────────────────────┘
                           │
                           ▼
                  ┌──────────────────┐
                  │  Alerting System  │
                  │                  │
                  │ latency > 2s ──▶ │──▶ PagerDuty
                  │ errors > 0  ──▶ │──▶ Slack
                  │ downtime    ──▶ │──▶ StatusPage
                  └──────────────────┘
```

### Deployment Verification Pipeline

Combine automated checks into a post-deployment verification pipeline that runs after every release.

```yaml
# .github/workflows/post-deploy-verify.yml
name: Post-Deployment Verification

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      target_url:
        required: true
        type: string

jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Wait for deployment to stabilize
        run: sleep 30

      - name: Run smoke tests
        run: |
          TARGET_HOST=${{ inputs.target_url }} \
          ./scripts/smoke-tests.sh

      - name: Check error rate
        run: |
          ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query" \
            --data-urlencode 'query=sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))' \
            | jq '.data.result[0].value[1] | tonumber')

          if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
            echo "ERROR: Error rate ${ERROR_RATE} exceeds threshold 0.01"
            exit 1
          fi

      - name: Check P99 latency
        run: |
          P99=$(curl -s "http://prometheus:9090/api/v1/query" \
            --data-urlencode 'query=histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))' \
            | jq '.data.result[0].value[1] | tonumber')

          if (( $(echo "$P99 > 2.0" | bc -l) )); then
            echo "ERROR: P99 latency ${P99}s exceeds threshold 2.0s"
            exit 1
          fi

      - name: Notify on success
        if: success()
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -H "Content-Type: application/json" \
            -d '{"text":"✅ Deployment to ${{ inputs.environment }} verified successfully"}'

      - name: Notify on failure
        if: failure()
        run: |
          curl -X POST "${{ secrets.SLACK_WEBHOOK }}" \
            -H "Content-Type: application/json" \
            -d '{"text":"❌ Deployment verification FAILED for ${{ inputs.environment }} — investigate immediately"}'
```

---

## Release Orchestration

Release orchestration coordinates the timing, versioning, and communication of software releases across teams and systems.

### Release Trains

A release train is a fixed-cadence release schedule — features that are ready board the train; features that are not wait for the next one. This decouples feature completion from release timing.

```
                          Release Train Schedule

    Week 1          Week 2          Week 3          Week 4
    ──────────────────────────────────────────────────────────▶ Time

    ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
    │ Train   │     │ Train   │     │ Train   │     │ Train   │
    │ v2.1    │     │ v2.2    │     │ v2.3    │     │ v2.4    │
    │         │     │         │     │         │     │         │
    │ Feat A ✓│     │ Feat C ✓│     │ Feat E ✓│     │ Feat B ✓│
    │ Feat B ✗│     │ Feat D ✓│     │ Feat F ✗│     │ Feat F ✓│
    │ Fix #42 ✓│     │ Fix #51 ✓│     │ Fix #58 ✓│     │ Fix #63 ✓│
    └─────────┘     └─────────┘     └─────────┘     └─────────┘
        │               │               │               │
    ✗ = not ready,  ✓ = included     Feature B and F    Feature F
    waits for next                   missed trains,     finally ready
    train                            caught later ones
```

**Benefits of release trains:**

- Predictable release cadence reduces coordination overhead
- Teams plan work around known release dates
- Missed features are not a crisis — they ship on the next train
- Stakeholders know exactly when to expect new capabilities

### Semantic Versioning

Semantic Versioning (SemVer) communicates the nature of changes through version numbers: `MAJOR.MINOR.PATCH`.

```
                    Semantic Versioning

                     v 2 . 3 . 1
                       │   │   │
                       │   │   └── PATCH: backward-compatible bug fixes
                       │   └────── MINOR: backward-compatible new features
                       └────────── MAJOR: breaking changes

    Examples:
    ─────────────────────────────────────────────────────
    1.0.0 → 1.0.1    Patch: fixed null pointer exception
    1.0.1 → 1.1.0    Minor: added new /search endpoint
    1.1.0 → 2.0.0    Major: removed deprecated /v1 API
    2.0.0-rc.1        Pre-release: release candidate 1
    2.0.0+build.123   Build metadata: CI build number
```

| Version Component | Increment When | Consumer Action |
|---|---|---|
| **MAJOR** (X.0.0) | Breaking API changes | Must update integration code |
| **MINOR** (0.X.0) | New features, backward-compatible | Can adopt at convenience |
| **PATCH** (0.0.X) | Bug fixes, backward-compatible | Should adopt promptly |
| **Pre-release** (X.Y.Z-alpha.1) | Testing before stable release | Opt-in only, not for production |

### Changelogs

A well-maintained changelog communicates what changed, why, and how users are affected. Automate changelog generation from commit messages or PR metadata.

```markdown
# Changelog

## [2.3.0] — 2025-03-15

### Added
- New /api/v2/search endpoint with full-text search support
- Rate limiting headers (X-RateLimit-Remaining, X-RateLimit-Reset)
- Health check endpoint now reports database connection pool status

### Changed
- Improved query performance for /api/v2/items (P99: 800ms → 200ms)
- Updated Redis client library from v4.2.0 to v4.5.1

### Fixed
- Fixed race condition in concurrent order processing (#451)
- Corrected timezone handling for scheduled reports (#462)

### Deprecated
- /api/v1/search endpoint — will be removed in v3.0.0

### Security
- Patched CVE-2025-1234 in image processing library
```

```yaml
# GitHub Actions — automated changelog generation
name: Release

on:
  push:
    tags:
      - "v*"

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Generate changelog
        id: changelog
        run: |
          PREVIOUS_TAG=$(git describe --tags --abbrev=0 HEAD^ 2>/dev/null || echo "")
          if [ -n "$PREVIOUS_TAG" ]; then
            CHANGELOG=$(git log ${PREVIOUS_TAG}..HEAD --pretty=format:"- %s (%h)" --no-merges)
          else
            CHANGELOG=$(git log --pretty=format:"- %s (%h)" --no-merges)
          fi
          echo "changelog<<EOF" >> "$GITHUB_OUTPUT"
          echo "$CHANGELOG" >> "$GITHUB_OUTPUT"
          echo "EOF" >> "$GITHUB_OUTPUT"

      - name: Create GitHub Release
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.repos.createRelease({
              owner: context.repo.owner,
              repo: context.repo.repo,
              tag_name: context.ref.replace('refs/tags/', ''),
              name: context.ref.replace('refs/tags/', ''),
              body: `${{ steps.changelog.outputs.changelog }}`,
              draft: false,
              prerelease: false
            })
```

---

## Next Steps

Continue your CI/CD learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | CI/CD Overview | Core CI/CD concepts, pipeline anatomy, and tool landscape |
| [01-CONTINUOUS-INTEGRATION.md](01-CONTINUOUS-INTEGRATION.md) | Continuous Integration | Build automation, testing, artifact management, and quality gates |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Continuous Delivery & Deployment documentation |
