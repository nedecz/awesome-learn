# CI/CD Best Practices

A comprehensive guide to CI/CD best practices for production — covering pipeline design, trunk-based development, build optimization, environment management, artifact and secret management, testing strategy, deployment patterns, observability, documentation, team practices, and operational readiness checklists.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Pipeline Design Principles](#pipeline-design-principles)
   - [Fast Feedback](#fast-feedback)
   - [Fail Fast](#fail-fast)
   - [Idempotent Builds](#idempotent-builds)
   - [Pipeline as Code](#pipeline-as-code)
3. [Trunk-Based Development](#trunk-based-development)
   - [Short-Lived Branches](#short-lived-branches)
   - [Merge Frequency](#merge-frequency)
   - [Branch Protection](#branch-protection)
4. [Build Optimization](#build-optimization)
   - [Caching](#caching)
   - [Parallelism](#parallelism)
   - [Incremental Builds](#incremental-builds)
   - [Build Matrix](#build-matrix)
5. [Environment Management](#environment-management)
   - [Environment Parity](#environment-parity)
   - [Infrastructure as Code](#infrastructure-as-code)
   - [Ephemeral Environments](#ephemeral-environments)
6. [Artifact Management](#artifact-management)
   - [Immutable Artifacts](#immutable-artifacts)
   - [Semantic Versioning](#semantic-versioning)
   - [Image Tagging Strategy](#image-tagging-strategy)
7. [Secret Management](#secret-management)
   - [Vault Integration](#vault-integration)
   - [Secret Rotation](#secret-rotation)
   - [Least Privilege](#least-privilege)
8. [Testing Strategy](#testing-strategy)
   - [Test Pyramid](#test-pyramid)
   - [Shift-Left Testing](#shift-left-testing)
   - [Quality Gates](#quality-gates)
   - [Flaky Test Management](#flaky-test-management)
9. [Deployment Patterns](#deployment-patterns)
   - [Blue-Green Deployments](#blue-green-deployments)
   - [Canary Deployments](#canary-deployments)
   - [Rolling Deployments](#rolling-deployments)
   - [Feature Flags](#feature-flags)
   - [Deployment Pattern Comparison](#deployment-pattern-comparison)
10. [Observability in Pipelines](#observability-in-pipelines)
    - [Build Metrics](#build-metrics)
    - [Deployment Frequency](#deployment-frequency)
    - [DORA Metrics](#dora-metrics)
11. [Documentation and Runbooks](#documentation-and-runbooks)
    - [Pipeline Documentation](#pipeline-documentation)
    - [Incident Playbooks](#incident-playbooks)
12. [Team Practices](#team-practices)
    - [Code Review Automation](#code-review-automation)
    - [ChatOps](#chatops)
    - [On-Call Integration](#on-call-integration)
13. [CI/CD Production Checklist](#cicd-production-checklist)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

Effective CI/CD is more than configuring a build server. Pipelines must be fast, reliable, and secure. They must build once and promote the same artifact across environments. They must provide rapid feedback, enforce quality gates, and integrate seamlessly with deployment strategies that minimize risk. Teams must treat pipeline definitions as production code — versioned, reviewed, tested, and documented.

This document distills CI/CD best practices from industry experience, the DORA research program, and production operations into actionable guidance for teams building and operating software delivery pipelines.

### Target Audience

- **Developers** writing code that flows through CI/CD pipelines and authoring pipeline definitions
- **DevOps Engineers** designing, building, and maintaining CI/CD pipelines across environments
- **SREs** ensuring deployment reliability, rollback mechanisms, and observability within pipelines
- **Platform Engineers** defining CI/CD standards and developer experience for organizations

### Scope

- Pipeline design, structure, and optimization
- Branching strategy and trunk-based development
- Build caching, parallelism, and artifact management
- Environment management and infrastructure as code
- Secret management and security in pipelines
- Testing strategy and quality gates
- Deployment patterns and release management
- Observability, documentation, and team practices

---

## Pipeline Design Principles

### Fast Feedback

The primary goal of a CI pipeline is to provide fast feedback to developers. If the pipeline is slow, developers will avoid running it — or worse, ignore its results.

```
Fast Feedback Loop
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Developer pushes code
       │
       ▼
  ┌──────────┐  < 1 min   ┌──────────┐  < 5 min   ┌──────────┐
  │  Lint &  │────────────▶│  Unit    │────────────▶│  Build   │
  │  Format  │             │  Tests   │             │  & Pack  │
  └──────────┘             └──────────┘             └──────────┘
       │                        │                        │
       ▼                        ▼                        ▼
  Instant feedback         Fast feedback           Build artifact
  on style issues          on logic errors         ready for deploy

  Target: Full CI pipeline completes in < 10 minutes
```

| Pipeline Stage | Target Duration | What It Catches |
|---|---|---|
| Lint & format | < 1 min | Style issues, syntax errors |
| Unit tests | < 5 min | Logic errors, regressions |
| Build & package | < 3 min | Compilation errors, dependency issues |
| Integration tests | < 10 min | Service interaction failures |
| Security scan | < 5 min | Vulnerabilities, secret leaks |
| **Total CI** | **< 10 min** | **All of the above** |

### Fail Fast

Order pipeline stages so the cheapest, fastest checks run first. If linting fails, there is no reason to run a 10-minute integration test suite.

```yaml
# .github/workflows/ci.yml — fail-fast pipeline
name: CI

on:
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint
      - run: npm run format:check

  unit-test:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test -- --coverage

  build:
    needs: unit-test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build

  integration-test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:integration
```

```
Fail-Fast Stage Ordering
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Lint (30s)  ──▶  Unit Tests (3m)  ──▶  Build (2m)  ──▶  Integration (8m)
      │                 │                     │                  │
      ✗ Fail            ✗ Fail                ✗ Fail             ✗ Fail
      │                 │                     │                  │
      ▼                 ▼                     ▼                  ▼
  Stop in 30s      Stop in 3.5m         Stop in 5.5m       Stop in 13.5m

  ❌ Bad: Run everything in parallel — waste 13 min on lint errors
  ✅ Good: Sequential gates — catch lint errors in 30 seconds
```

### Idempotent Builds

The same commit must produce the same artifact every time. Non-deterministic builds create debugging nightmares and undermine trust in the pipeline.

```yaml
# ✅ Good: Pinned dependencies — reproducible
- run: npm ci                          # Uses exact versions from lockfile
- run: pip install -r requirements.txt # Pinned versions (==)
- run: go build -trimpath ./...        # Reproducible Go binary

# ❌ Bad: Floating dependencies — non-reproducible
- run: npm install                     # May resolve different versions
- run: pip install flask               # Installs latest, unpinned
- run: curl -sSL https://get.tool.io | bash  # Version changes over time
```

| Practice | Why It Matters |
|---|---|
| Pin all dependency versions | Same input → same output |
| Use lockfiles (`package-lock.json`, `go.sum`, `poetry.lock`) | Deterministic dependency resolution |
| Pin CI runner images and tool versions | Runner changes do not break builds |
| Avoid `latest` tags for base images | Base image changes do not break builds |
| Use `--frozen-lockfile` or `npm ci` | Fail if lockfile is out of date |
| Set deterministic build flags (`-trimpath`, `SOURCE_DATE_EPOCH`) | Binary reproducibility |

### Pipeline as Code

Define pipelines in version-controlled files alongside application code. Never configure pipelines through a UI alone.

```
Pipeline as Code Benefits
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────────────────────────────────────┐
  │  my-service/                                │
  │  ├── src/                                   │
  │  ├── tests/                                 │
  │  ├── Dockerfile                             │
  │  ├── .github/workflows/ci.yml      ◀── CI  │
  │  ├── .github/workflows/deploy.yml  ◀── CD  │
  │  └── infrastructure/                        │
  │      ├── terraform/                         │
  │      └── helm/                              │
  └─────────────────────────────────────────────┘

  ✅ Versioned: Pipeline changes go through code review
  ✅ Auditable: Git history shows who changed what and when
  ✅ Reproducible: Any commit can recreate the exact pipeline
  ✅ Testable: Pipeline logic can be validated before merge
```

---

## Trunk-Based Development

### Short-Lived Branches

Keep feature branches short-lived (ideally < 1 day, at most 2-3 days). Long-lived branches cause merge conflicts, integration problems, and delayed feedback.

```
Short-Lived Branches
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ✅ Good: Short-lived branches (hours to 1-2 days)

  main ─────●─────●─────●─────●─────●─────●───▶
             ╲   ╱       ╲   ╱       ╲   ╱
  feat-A      ●─●         │  │        │  │
  feat-B                  ●──●        │  │
  feat-C                              ●──●


  ❌ Bad: Long-lived branches (weeks to months)

  main ─────●───────────────────────────────●───▶
             ╲                             ╱
  feature     ●───●───●───●───●───●───●───●
              (3 weeks of divergence = merge nightmare)
```

| Metric | Target | Why |
|---|---|---|
| Branch lifetime | < 2 days | Reduces merge conflicts |
| Lines changed per PR | < 400 | Easier to review |
| PRs merged per dev per day | ≥ 1 | Continuous integration, not "periodic" integration |
| Time from open to merge | < 24 hours | Fast flow, short feedback loop |

### Merge Frequency

Merge to trunk frequently. The longer code lives outside of trunk, the higher the integration risk.

```yaml
# Branch policy: Enforce frequent merges
# Azure DevOps example
- policy:
    type: branch-freshness
    settings:
      maxDaysStale: 2
      message: "Branch is stale. Rebase on main and merge or close."
```

```
Integration Risk vs. Branch Age
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Risk │
       │                              ╱
       │                           ╱
       │                        ╱
       │                     ╱
       │                 ╱
       │             ╱
       │         .─'
       │      .─'
       │   .─'
       │ .─'
       └──────────────────────────────
         1d   2d   3d   1w   2w   1m
                Branch Age

  Risk grows non-linearly with branch age.
  A 2-week branch is not 2x harder than a 1-week branch — it is 5-10x harder.
```

### Branch Protection

Protect the trunk (main) branch with required checks and reviews.

```yaml
# GitHub branch protection (as code via API or Terraform)
# Required for main branch:
branch_protection:
  required_status_checks:
    strict: true                    # Branch must be up to date
    contexts:
      - "ci/lint"
      - "ci/test"
      - "ci/build"
      - "security/scan"
  required_pull_request_reviews:
    required_approving_review_count: 1
    dismiss_stale_reviews: true
    require_code_owner_reviews: true
  enforce_admins: true              # No bypassing for admins
  restrictions: null                # All contributors can push
  required_linear_history: true     # No merge commits
```

| Protection Rule | Purpose |
|---|---|
| Required status checks | Pipeline must pass before merge |
| Require up-to-date branch | Prevents merging stale code |
| Required reviews | Human verification of changes |
| Dismiss stale reviews | Re-review after new pushes |
| CODEOWNERS review | Domain experts approve their areas |
| Linear history | Clean, bisectable history |
| No force push | Prevent history rewriting on trunk |

---

## Build Optimization

### Caching

Cache dependencies and build outputs to avoid redundant work. Caching is the single most effective way to speed up CI pipelines.

```yaml
# GitHub Actions — dependency caching
- name: Cache node modules
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

- name: Install dependencies
  run: npm ci
```

```yaml
# GitHub Actions — Go module caching
- name: Cache Go modules
  uses: actions/cache@v4
  with:
    path: |
      ~/go/pkg/mod
      ~/.cache/go-build
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
    restore-keys: |
      ${{ runner.os }}-go-
```

```yaml
# GitHub Actions — Docker layer caching with Buildx
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build with cache
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: myregistry.io/myapp:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

| Cache Target | Key Strategy | Invalidation |
|---|---|---|
| npm / yarn / pnpm | Hash of `package-lock.json` | Lockfile changes |
| pip / poetry | Hash of `requirements.txt` / `poetry.lock` | Dependency changes |
| Go modules | Hash of `go.sum` | Module changes |
| Docker layers | BuildKit `type=gha` or `type=registry` | Dockerfile or context changes |
| Gradle / Maven | Hash of `build.gradle` / `pom.xml` | Build config changes |
| Compiled output | Hash of source files | Source changes |

### Parallelism

Run independent stages and tests in parallel to reduce total pipeline duration.

```yaml
# GitHub Actions — parallel jobs
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run lint

  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high

  # Gate: all parallel jobs must pass before build
  build:
    needs: [lint, unit-test, security-scan]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
```

```
Sequential vs. Parallel Execution
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ❌ Sequential (total: 18 min)
  ┌──────┐ ┌──────────┐ ┌────────┐ ┌────────────────┐
  │ Lint │→│Unit Tests│→│Security│→│     Build      │
  │ 2min │ │  5min    │ │ 3min   │ │     8min       │
  └──────┘ └──────────┘ └────────┘ └────────────────┘

  ✅ Parallel (total: 13 min)
  ┌──────┐
  │ Lint │──────┐
  │ 2min │      │
  └──────┘      │
  ┌──────────┐  ├──▶ ┌────────────────┐
  │Unit Tests│──┘    │     Build      │
  │  5min    │──┐    │     8min       │
  └──────────┘  │    └────────────────┘
  ┌────────┐    │
  │Security│────┘
  │ 3min   │
  └────────┘
  Wait for longest parallel job (5 min) + Build (8 min) = 13 min
```

### Incremental Builds

Only build and test what changed. In monorepos, this is critical for keeping pipeline times manageable.

```yaml
# GitHub Actions — path-based triggers
on:
  push:
    paths:
      - 'services/api/**'
      - 'libs/shared/**'
      - '.github/workflows/api.yml'

# Only runs when API service or shared libs change
jobs:
  build-api:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cd services/api && npm ci && npm run build
```

```yaml
# Monorepo — detect changed services
- name: Detect changes
  id: changes
  uses: dorny/paths-filter@v3
  with:
    filters: |
      api:
        - 'services/api/**'
      web:
        - 'services/web/**'
      shared:
        - 'libs/shared/**'

- name: Build API
  if: steps.changes.outputs.api == 'true' || steps.changes.outputs.shared == 'true'
  run: cd services/api && npm ci && npm run build

- name: Build Web
  if: steps.changes.outputs.web == 'true' || steps.changes.outputs.shared == 'true'
  run: cd services/web && npm ci && npm run build
```

### Build Matrix

Use build matrices to test across multiple platforms, language versions, or configurations in parallel.

```yaml
# GitHub Actions — build matrix
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [18, 20, 22]
        exclude:
          - os: windows-latest
            node-version: 18
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci && npm test
```

| Matrix Strategy | Use Case | Example |
|---|---|---|
| OS matrix | Cross-platform compatibility | `[ubuntu, windows, macos]` |
| Language version matrix | Version compatibility | `[node 18, 20, 22]` |
| Database matrix | Multi-DB support | `[postgres 14, 15, 16]` |
| `fail-fast: false` | See all failures, not just first | Full compatibility report |
| `exclude` | Skip invalid combinations | Windows + unsupported version |
| `include` | Add specific extra combos | ARM + specific version |

---

## Environment Management

### Environment Parity

Keep development, staging, and production environments as similar as possible. Differences between environments are a top cause of "works on my machine" and deployment failures.

```
Environment Parity
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ✅ Good: Same stack, same config shape, same image

  ┌───────────┐    ┌───────────┐    ┌───────────┐
  │    Dev    │    │  Staging  │    │   Prod    │
  ├───────────┤    ├───────────┤    ├───────────┤
  │ Image:    │    │ Image:    │    │ Image:    │
  │ app:abc12 │    │ app:abc12 │    │ app:abc12 │
  │ PG 16     │    │ PG 16     │    │ PG 16     │
  │ Redis 7   │    │ Redis 7   │    │ Redis 7   │
  │ K8s 1.29  │    │ K8s 1.29  │    │ K8s 1.29  │
  └───────────┘    └───────────┘    └───────────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
              Only differences:
              • Scale (replicas, resources)
              • Secrets (credentials)
              • DNS / endpoints
```

| Dimension | Must Match | Can Differ |
|---|---|---|
| Application image | ✅ Same SHA | |
| OS and runtime version | ✅ Same version | |
| Database version | ✅ Same major version | |
| Configuration shape | ✅ Same keys | Values differ per env |
| Secrets | | ✅ Different per env |
| Scale (replicas, CPU, RAM) | | ✅ Different per env |
| DNS / endpoints | | ✅ Different per env |

### Infrastructure as Code

Define all infrastructure through code. Never configure environments through manual console clicks.

```yaml
# Terraform — define environment infrastructure
resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  dns_prefix          = "app-${var.environment}"

  default_node_pool {
    name       = "default"
    node_count = var.node_count
    vm_size    = var.vm_size
  }

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

```yaml
# Environment-specific variables
# terraform/environments/staging.tfvars
environment    = "staging"
node_count     = 2
vm_size        = "Standard_D2s_v3"

# terraform/environments/production.tfvars
environment    = "production"
node_count     = 5
vm_size        = "Standard_D4s_v3"
```

| IaC Principle | Practice |
|---|---|
| Version controlled | All infra definitions in Git |
| Reviewed | Infra changes go through PR review |
| Tested | Use `terraform plan` / `terraform validate` in CI |
| Idempotent | Running twice produces the same result |
| Modular | Reusable modules for common patterns |
| State managed | Remote state with locking (S3 + DynamoDB, Azure Blob) |

### Ephemeral Environments

Spin up temporary environments for pull requests and tear them down after merge. This provides isolated testing without maintaining long-lived staging environments.

```yaml
# GitHub Actions — ephemeral preview environment
name: Preview Environment

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy preview
        run: |
          helm upgrade --install preview-${{ github.event.number }} ./chart \
            --namespace previews \
            --set image.tag=${{ github.sha }} \
            --set ingress.host=pr-${{ github.event.number }}.preview.example.com

      - name: Comment PR with preview URL
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `Preview deployed: https://pr-${context.issue.number}.preview.example.com`
            })

  cleanup-preview:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - run: |
          helm uninstall preview-${{ github.event.number }} \
            --namespace previews
```

---

## Artifact Management

### Immutable Artifacts

Build once, deploy everywhere. Never rebuild the same code for different environments. The artifact deployed to production must be the exact same binary tested in staging.

```
Build Once, Deploy Everywhere
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Source Code
       │
       ▼
  ┌──────────┐      ┌──────────────┐
  │  Build   │─────▶│   Registry   │
  │  (once)  │      │  (immutable) │
  └──────────┘      └──────┬───────┘
                           │
                    Same artifact SHA
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         ┌────────┐  ┌────────┐  ┌────────┐
         │  Dev   │  │Staging │  │  Prod  │
         └────────┘  └────────┘  └────────┘

  Configuration differences via environment variables,
  config files, or secret injection — NOT different builds.
```

```yaml
# ❌ Bad: Rebuilding per environment
jobs:
  build-staging:
    steps:
      - run: docker build --build-arg ENV=staging -t app:staging .
  build-production:
    steps:
      - run: docker build --build-arg ENV=production -t app:production .

# ✅ Good: Build once, promote across environments
jobs:
  build:
    steps:
      - run: docker build -t myregistry.io/app:${{ github.sha }} .
      - run: docker push myregistry.io/app:${{ github.sha }}

  deploy-staging:
    needs: build
    steps:
      - run: helm upgrade app ./chart --set image.tag=${{ github.sha }}

  deploy-production:
    needs: deploy-staging
    steps:
      - run: helm upgrade app ./chart --set image.tag=${{ github.sha }}
```

### Semantic Versioning

Tag releases with semantic versions for clear communication about the nature of changes.

```
Semantic Versioning: MAJOR.MINOR.PATCH
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  v2.4.1
  │ │ │
  │ │ └── PATCH: Bug fixes, no API changes
  │ └──── MINOR: New features, backward compatible
  └────── MAJOR: Breaking changes
```

```yaml
# Automated versioning in CI based on conventional commits
# Uses commit message prefixes: fix:, feat:, BREAKING CHANGE:
- name: Determine version bump
  id: version
  run: |
    COMMITS=$(git log --oneline ${{ github.event.before }}..${{ github.sha }})
    if echo "$COMMITS" | grep -q "BREAKING CHANGE"; then
      echo "bump=major" >> "$GITHUB_OUTPUT"
    elif echo "$COMMITS" | grep -q "^feat"; then
      echo "bump=minor" >> "$GITHUB_OUTPUT"
    else
      echo "bump=patch" >> "$GITHUB_OUTPUT"
    fi
```

| Commit Message | Version Bump | Example |
|---|---|---|
| `fix: correct null check` | Patch | `1.2.3` → `1.2.4` |
| `feat: add search endpoint` | Minor | `1.2.4` → `1.3.0` |
| `feat!: redesign auth API` | Major | `1.3.0` → `2.0.0` |

### Image Tagging Strategy

Use multiple tags for traceability and convenience.

```bash
# Tag with Git SHA (immutable, traceable)
docker tag app:local myregistry.io/app:abc1234

# Tag with semver (human-readable)
docker tag app:local myregistry.io/app:v1.2.3

# Tag with both (best practice)
docker tag app:local myregistry.io/app:v1.2.3-abc1234
```

| Tag Type | Example | Traceability | Mutable | Use In Production |
|---|---|---|---|---|
| Git SHA | `abc1234` | Exact commit | No | ✅ Yes |
| Semver | `v1.2.3` | Release version | No | ✅ Yes |
| Semver + SHA | `v1.2.3-abc1234` | Both | No | ✅ Best |
| Branch | `main` | Current branch tip | Yes | ❌ No |
| `latest` | `latest` | None | Yes | ❌ Never |

---

## Secret Management

### Vault Integration

Never store secrets in pipeline code, environment variables baked into images, or Git repositories. Use a dedicated secrets manager.

```yaml
# GitHub Actions — using GitHub Secrets
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Login to registry
        uses: docker/login-action@v3
        with:
          registry: myregistry.io
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}
```

```yaml
# GitHub Actions — fetching secrets from Azure Key Vault
- name: Azure Login
  uses: azure/login@v2
  with:
    creds: ${{ secrets.AZURE_CREDENTIALS }}

- name: Get secrets from Key Vault
  uses: azure/get-keyvault-secrets@v1
  with:
    keyvault: "my-keyvault"
    secrets: "db-password, api-key"
  id: secrets

- name: Use secret
  run: |
    # Secret is masked in logs automatically
    echo "Deploying with DB connection..."
    helm upgrade app ./chart \
      --set db.password=${{ steps.secrets.outputs.db-password }}
```

| Secrets Manager | Platform | Integration |
|---|---|---|
| GitHub Secrets | GitHub Actions | Native, environment-scoped |
| Azure Key Vault | Azure / any CI | Azure CLI, GitHub Action |
| HashiCorp Vault | Any | API, CLI, CI plugins |
| AWS Secrets Manager | AWS / any CI | AWS CLI, GitHub Action |
| GCP Secret Manager | GCP / any CI | gcloud CLI, GitHub Action |

### Secret Rotation

Automate secret rotation so compromised credentials have limited blast radius.

```yaml
# Rotation policy
secret_rotation:
  database_passwords:
    rotation_interval: 30 days
    method: automated
    notification: team-security-channel

  api_keys:
    rotation_interval: 90 days
    method: automated
    notification: team-platform-channel

  tls_certificates:
    rotation_interval: 365 days
    method: cert-manager auto-renewal
    alert_before_expiry: 30 days
```

| Practice | Description |
|---|---|
| Automate rotation | Manual rotation is error-prone and forgotten |
| Use short-lived tokens | OIDC tokens > static credentials |
| Alert before expiry | Detect expiring secrets before they break production |
| Audit access logs | Know who accessed which secret and when |
| Use environment-scoped secrets | Staging secrets ≠ production secrets |

### Least Privilege

Grant pipelines only the permissions they need. A CI job that runs linting does not need production deployment credentials.

```yaml
# GitHub Actions — minimal permissions per job
jobs:
  lint:
    runs-on: ubuntu-latest
    permissions:
      contents: read             # Only read code
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint

  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read             # Read code
      packages: write            # Push images
      id-token: write            # OIDC for cloud auth
    environment: production      # Requires approval
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        run: ./deploy.sh
```

```
Least Privilege Pipeline Permissions
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ❌ Bad: All jobs share admin-level service account

  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
  │ Lint │  │ Test │  │Build │  │Deploy│
  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘
     └──────────┴──────────┴──────────┘
                    │
            ┌──────────────┐
            │  Admin Creds │  ← Every job can deploy to prod!
            └──────────────┘

  ✅ Good: Each job has only the permissions it needs

  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
  │ Lint │  │ Test │  │Build │  │Deploy│
  └──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘
     │         │         │         │
  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────────┐
  │ Read │  │ Read │  │ Read │  │  Deploy   │
  │ Only │  │ Only │  │+Push │  │  + OIDC   │
  └──────┘  └──────┘  └──────┘  └──────────┘
```

---

## Testing Strategy

### Test Pyramid

Structure your tests as a pyramid — many fast unit tests, fewer integration tests, and even fewer end-to-end tests.

```
Test Pyramid
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

          ╱╲
         ╱  ╲           E2E Tests
        ╱ E2E╲          • Slow (minutes)
       ╱      ╲         • Fragile
      ╱────────╲        • Few (5-10%)
     ╱          ╲
    ╱Integration ╲     Integration Tests
   ╱              ╲     • Medium (seconds)
  ╱────────────────╲    • Some (20-30%)
 ╱                  ╲
╱    Unit Tests      ╲  Unit Tests
╲                    ╱   • Fast (milliseconds)
 ╲──────────────────╱    • Stable
                         • Many (60-70%)

  Cost increases ▲
  Speed decreases ▲
  Coverage breadth increases ▲
```

| Test Type | Speed | Scope | Count | Run In CI |
|---|---|---|---|---|
| **Unit** | ms | Single function / class | 1000s | Every commit |
| **Integration** | sec | Service + dependencies | 100s | Every commit |
| **E2E** | min | Full system flow | 10s | PR merge, nightly |
| **Performance** | min | Load and latency | Few | Nightly, pre-release |
| **Security** | min | Vulnerability scan | Few | Every commit |

### Shift-Left Testing

Run tests as early as possible in the development lifecycle. Bugs caught in development are 10-100x cheaper to fix than bugs found in production.

```
Shift-Left: Move Testing Earlier
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Traditional (shift-right):
  Code ──▶ Build ──▶ Deploy ──▶ QA ──▶ Staging ──▶ Prod
                                 ▲
                            Tests here (late, expensive)

  Shift-left:
  Code ──▶ Build ──▶ Deploy ──▶ Staging ──▶ Prod
   ▲         ▲         ▲
   │         │         └── Integration tests
   │         └── SAST, dependency scan
   └── Lint, unit tests, pre-commit hooks

  Bugs found earlier = cheaper to fix
```

```yaml
# Pre-commit hooks — catch issues before they enter the pipeline
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.3.0
    hooks:
      - id: ruff
      - id: ruff-format
```

### Quality Gates

Define mandatory quality checks that must pass before code can progress through the pipeline.

```yaml
# Quality gate configuration
quality_gates:
  pr_merge:
    - lint: pass
    - unit_tests: pass
    - coverage: ">= 80%"
    - security_scan: "no critical/high"
    - pr_review: ">= 1 approval"

  staging_deploy:
    - integration_tests: pass
    - performance_test: "p99 < 200ms"
    - staging_smoke_test: pass

  production_deploy:
    - staging_validation: "24h soak"
    - change_approval: approved
    - deployment_window: "business hours"
    - rollback_plan: documented
```

```yaml
# GitHub Actions — coverage quality gate
- name: Run tests with coverage
  run: npm test -- --coverage --coverageReporters=json-summary

- name: Check coverage threshold
  run: |
    COVERAGE=$(jq '.total.lines.pct' coverage/coverage-summary.json)
    echo "Line coverage: ${COVERAGE}%"
    if (( $(echo "$COVERAGE < 80" | bc -l) )); then
      echo "❌ Coverage ${COVERAGE}% is below 80% threshold"
      exit 1
    fi
    echo "✅ Coverage ${COVERAGE}% meets threshold"
```

| Quality Gate | Stage | Threshold |
|---|---|---|
| Lint / format | PR | Must pass |
| Unit test coverage | PR | ≥ 80% lines |
| No critical vulnerabilities | PR | Zero critical / high |
| PR approval | PR | ≥ 1 reviewer |
| Integration tests | Pre-deploy | Must pass |
| Performance regression | Pre-deploy | p99 < baseline + 10% |
| Smoke tests | Post-deploy | Must pass |

### Flaky Test Management

Flaky tests erode trust in the pipeline. If developers learn to ignore failures, real bugs slip through.

```yaml
# Strategy: Quarantine flaky tests
# 1. Detect: Track test pass rates over time
# 2. Quarantine: Move flaky tests to a separate suite
# 3. Fix or delete: Flaky tests have a 2-week SLA to fix

# Jest example — mark flaky tests
describe.skip('flaky: payment processing', () => {
  // TODO: Fix by 2025-02-15 — see JIRA-1234
  // Flaky due to third-party API timeout
});
```

```yaml
# GitHub Actions — retry flaky tests before failing
- name: Run tests with retry
  uses: nick-fields/retry@v3
  with:
    timeout_minutes: 10
    max_attempts: 3
    retry_on: error
    command: npm test
```

| Practice | Description |
|---|---|
| Track flake rate | Monitor which tests fail intermittently |
| Quarantine, do not skip | Move to separate suite; still run nightly |
| Set fix SLA | 2-week deadline to fix or delete |
| Retry with caution | Auto-retry hides flakiness — use sparingly |
| Never ignore failures | A red pipeline must always mean "stop and investigate" |

---

## Deployment Patterns

### Blue-Green Deployments

Maintain two identical production environments. Deploy to the inactive one, then switch traffic.

```
Blue-Green Deployment
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Step 1: Blue is live, deploy to Green
                   ┌──────────────┐
  Users ──▶ LB ──▶│  Blue (v1)   │ ◀── Live
                   └──────────────┘
                   ┌──────────────┐
                   │  Green (v2)  │ ◀── Deploy + test here
                   └──────────────┘

  Step 2: Switch traffic to Green
                   ┌──────────────┐
                   │  Blue (v1)   │ ◀── Standby (instant rollback)
                   └──────────────┘
  Users ──▶ LB ──▶┌──────────────┐
                   │  Green (v2)  │ ◀── Live
                   └──────────────┘

  Rollback: Switch LB back to Blue (seconds)
```

### Canary Deployments

Route a small percentage of traffic to the new version. Monitor for errors, then gradually increase.

```
Canary Deployment
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Step 1: 5% canary           Step 2: 25% canary
  ┌──────┐  95%  ┌────────┐   ┌──────┐  75%  ┌────────┐
  │      │──────▶│  v1    │   │      │──────▶│  v1    │
  │  LB  │       └────────┘   │  LB  │       └────────┘
  │      │  5%   ┌────────┐   │      │  25%  ┌────────┐
  │      │──────▶│  v2    │   │      │──────▶│  v2    │
  └──────┘       └────────┘   └──────┘       └────────┘
        Monitor error rate,          If healthy,
        latency, saturation          increase traffic

  Step 3: 100% promoted       Rollback: Route 100% back to v1
  ┌──────┐ 100%  ┌────────┐   (takes seconds, automatic on error)
  │  LB  │──────▶│  v2    │
  └──────┘       └────────┘
```

```yaml
# Kubernetes — canary with Argo Rollouts
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
        - setWeight: 25
        - pause: { duration: 10m }
        - setWeight: 50
        - pause: { duration: 10m }
        - setWeight: 100
      canaryMetadata:
        labels:
          role: canary
      stableMetadata:
        labels:
          role: stable
```

### Rolling Deployments

Replace instances one at a time (or in batches). The simplest strategy with zero additional infrastructure.

```yaml
# Kubernetes — rolling update (default strategy)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # One extra pod during update
      maxUnavailable: 0    # Never reduce below desired count
  template:
    spec:
      containers:
        - name: myapp
          image: myregistry.io/app:v2.0.0
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
```

### Feature Flags

Decouple deployment from release. Deploy code to production with features hidden behind flags, then enable them independently.

```
Feature Flags: Deploy ≠ Release
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Traditional:  Deploy v2 ──▶ Feature is live for all users
                (risky, all-or-nothing)

  Feature flags: Deploy v2 ──▶ Feature hidden ──▶ Enable for 5%
                                                 ──▶ Enable for 50%
                                                 ──▶ Enable for 100%
                (safe, gradual, instant rollback)

  ┌──────────────────────────────────────┐
  │  Code deployed to production:        │
  │                                      │
  │  if feature_flags.enabled("new_ui"): │
  │      show_new_ui()                   │
  │  else:                               │
  │      show_old_ui()                   │
  │                                      │
  │  Flag: "new_ui"                      │
  │  ├── Off for everyone (default)      │
  │  ├── On for internal team            │
  │  ├── On for 10% of users             │
  │  └── On for 100% (full rollout)      │
  └──────────────────────────────────────┘
```

### Deployment Pattern Comparison

| Pattern | Downtime | Rollback Speed | Infrastructure Cost | Risk | Complexity |
|---|---|---|---|---|---|
| **Blue-Green** | Zero | Instant (switch LB) | 2x (two full environments) | Low | Medium |
| **Canary** | Zero | Fast (route to stable) | 1x + canary instances | Very low | High |
| **Rolling** | Zero | Slow (roll back one by one) | 1x + surge buffer | Medium | Low |
| **Recreate** | Yes | Slow (redeploy previous) | 1x | High | Very low |
| **Feature Flags** | Zero | Instant (toggle flag) | 1x | Very low | Medium |

---

## Observability in Pipelines

### Build Metrics

Track pipeline performance to identify bottlenecks and measure improvement.

```yaml
# Key pipeline metrics to track
pipeline_metrics:
  - name: build_duration_seconds
    type: histogram
    description: "Time to complete CI pipeline"
    labels: [repo, branch, status]

  - name: build_queue_wait_seconds
    type: histogram
    description: "Time waiting for a runner"
    labels: [repo, runner_type]

  - name: test_pass_rate
    type: gauge
    description: "Percentage of tests passing"
    labels: [repo, test_suite]

  - name: flaky_test_count
    type: gauge
    description: "Number of quarantined flaky tests"
    labels: [repo]
```

| Metric | Target | Alert Threshold |
|---|---|---|
| CI pipeline duration | < 10 min | > 15 min |
| Queue wait time | < 1 min | > 5 min |
| Test pass rate | 100% | < 99% |
| Build success rate | > 95% | < 90% |
| Cache hit rate | > 80% | < 50% |

### Deployment Frequency

Track how often you deploy to production. High deployment frequency correlates with high-performing teams (per DORA research).

```
Deployment Frequency Maturity
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Elite       │ Multiple deploys per day
  ────────────┤
  High        │ Between once per day and once per week
  ────────────┤
  Medium      │ Between once per week and once per month
  ────────────┤
  Low         │ Between once per month and once per six months
  ────────────┘

  Goal: Move toward elite — many small deploys are safer
  than infrequent large deploys.
```

### DORA Metrics

The four DORA (DevOps Research and Assessment) metrics measure software delivery performance. Track all four to understand your team's effectiveness.

```
DORA Metrics
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────────────┐    ┌─────────────────────┐
  │  Deployment         │    │  Lead Time for      │
  │  Frequency          │    │  Changes            │
  │                     │    │                     │
  │  How often do you   │    │  How long from      │
  │  deploy to prod?    │    │  commit to prod?    │
  │                     │    │                     │
  │  Elite: on-demand   │    │  Elite: < 1 hour    │
  │  (multiple/day)     │    │  High:  < 1 day     │
  └─────────────────────┘    └─────────────────────┘

  ┌─────────────────────┐    ┌─────────────────────┐
  │  Change Failure     │    │  Time to Restore    │
  │  Rate               │    │  Service            │
  │                     │    │                     │
  │  What % of deploys  │    │  How long to        │
  │  cause failures?    │    │  recover from       │
  │                     │    │  failure?            │
  │  Elite: < 5%        │    │  Elite: < 1 hour    │
  │  High:  < 15%       │    │  High:  < 1 day     │
  └─────────────────────┘    └─────────────────────┘
```

| DORA Metric | Elite | High | Medium | Low |
|---|---|---|---|---|
| **Deployment Frequency** | On-demand (multiple/day) | Weekly to daily | Monthly to weekly | Monthly to biannual |
| **Lead Time for Changes** | Less than 1 hour | 1 day to 1 week | 1 week to 1 month | 1 month to 6 months |
| **Change Failure Rate** | 0-5% | 5-15% | 15-30% | 30-45% |
| **Time to Restore Service** | Less than 1 hour | Less than 1 day | 1 day to 1 week | 1 week to 1 month |

---

## Documentation and Runbooks

### Pipeline Documentation

Document your pipeline so anyone on the team can understand, debug, and modify it.

```markdown
# Pipeline Documentation Template

## Pipeline Overview
- **Repository**: github.com/org/my-service
- **CI Platform**: GitHub Actions
- **Trigger**: Push to main, PRs to main
- **Average Duration**: ~8 minutes

## Stages
1. **Lint** (1 min) — ESLint, Prettier
2. **Unit Tests** (3 min) — Jest, 92% coverage
3. **Build** (2 min) — Docker multi-stage build
4. **Security Scan** (2 min) — Trivy, npm audit
5. **Deploy to Staging** (3 min) — Helm upgrade
6. **Integration Tests** (5 min) — Against staging
7. **Deploy to Production** (3 min) — Helm upgrade, canary

## Required Secrets
| Secret | Source | Rotation |
|---|---|---|
| REGISTRY_TOKEN | Azure Key Vault | 90 days |
| KUBE_CONFIG | GitHub Environment | Per-cluster |
| SONAR_TOKEN | SonarCloud | Annual |

## Troubleshooting
- **Lint fails**: Run `npm run lint:fix` locally
- **Tests fail**: Check test output; run `npm test` locally
- **Build fails**: Check Dockerfile; verify base image availability
- **Deploy fails**: Check Kubernetes events; verify image exists in registry
```

### Incident Playbooks

Create runbooks for common deployment failures so on-call engineers can respond quickly.

```markdown
# Deployment Rollback Playbook

## Symptoms
- Error rate spike after deployment
- Latency p99 > 500ms (baseline: 150ms)
- Health check failures on new pods

## Immediate Actions (< 5 minutes)
1. **Confirm the issue**:
   kubectl get pods -n production -l app=myapp
   kubectl logs -n production -l app=myapp --tail=50
2. **Roll back**:
   kubectl rollout undo deployment/myapp -n production
3. **Verify rollback**:
   kubectl rollout status deployment/myapp -n production
4. **Notify team**: Post in #incidents channel

## Follow-Up (< 1 hour)
1. Create incident ticket
2. Gather logs and metrics from deployment window
3. Identify root cause
4. Write post-mortem if customer-impacting

## Escalation
- P1 (service down): Page on-call SRE immediately
- P2 (degraded): Notify on-call in Slack
- P3 (minor): Create ticket for next sprint
```

| Runbook | When to Use |
|---|---|
| Deployment rollback | Error rate spikes post-deploy |
| Pipeline failure triage | CI build is red on main |
| Secret rotation emergency | Secret leaked or compromised |
| Dependency vulnerability | Critical CVE in production dependency |
| Capacity scaling | Resource exhaustion, pod evictions |

---

## Team Practices

### Code Review Automation

Automate what machines do better than humans. Let reviewers focus on design, logic, and correctness.

```yaml
# GitHub Actions — automated PR checks
name: PR Automation

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  auto-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Automated formatting check
      - run: npm ci && npm run format:check

      # PR size label
      - uses: codelytv/pr-size-labeler@v1
        with:
          xs_max_size: 10
          s_max_size: 100
          m_max_size: 400
          l_max_size: 800

      # Auto-assign reviewers based on CODEOWNERS
      - uses: necojackarc/auto-request-review@v0.13.0
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
```

| Automate | Human Review |
|---|---|
| Formatting and style | Architecture decisions |
| Lint rules | Business logic correctness |
| Test coverage thresholds | Error handling completeness |
| Dependency vulnerability scan | API design and naming |
| PR size labeling | Performance implications |
| CODEOWNERS assignment | Security-sensitive changes |

### ChatOps

Bring deployment and pipeline operations into team chat for visibility and collaboration.

```
ChatOps Workflow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Developer in Slack:
  ┌─────────────────────────────────────────────┐
  │ @deploy-bot deploy myapp v1.2.3 to staging  │
  └───────────────────────────┬─────────────────┘
                              │
                              ▼
  ┌─────────────────────────────────────────────┐
  │ 🚀 Deploying myapp v1.2.3 to staging...    │
  │ ✅ Image found: myregistry.io/app:v1.2.3   │
  │ ✅ Helm upgrade successful                  │
  │ ✅ Health checks passing                    │
  │ 🎉 Deploy complete in 2m 15s               │
  └─────────────────────────────────────────────┘

  Team sees: who deployed what, when, and whether it worked.
```

```yaml
# GitHub Actions — Slack notification on deploy
- name: Notify Slack
  if: always()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "${{ job.status == 'success' && '✅' || '❌' }} Deploy ${{ github.sha }} to ${{ inputs.environment }} — ${{ job.status }}",
        "channel": "#deployments"
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### On-Call Integration

Connect your deployment pipeline to incident management so failed deployments automatically create alerts.

```yaml
# GitHub Actions — alert on production deploy failure
- name: Alert on failure
  if: failure() && github.ref == 'refs/heads/main'
  run: |
    curl -X POST "${{ secrets.PAGERDUTY_EVENTS_URL }}" \
      -H "Content-Type: application/json" \
      -d '{
        "routing_key": "${{ secrets.PAGERDUTY_ROUTING_KEY }}",
        "event_action": "trigger",
        "payload": {
          "summary": "Production deploy failed: ${{ github.repository }}",
          "severity": "critical",
          "source": "github-actions",
          "custom_details": {
            "run_url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}",
            "commit": "${{ github.sha }}",
            "author": "${{ github.actor }}"
          }
        }
      }'
```

| Integration | Purpose |
|---|---|
| Slack / Teams notifications | Deploy visibility for the team |
| PagerDuty / Opsgenie alerts | Page on-call for failed deploys |
| Jira / Linear tickets | Auto-create tickets for deploy failures |
| Grafana annotations | Mark deploys on dashboards for correlation |
| Status page updates | Communicate maintenance windows to users |

---

## CI/CD Production Checklist

| Category | Check | Priority | Status |
|---|---|---|---|
| **Pipeline** | Pipeline defined as code (YAML in repo) | 🔴 Critical | ☐ |
| **Pipeline** | Stages ordered for fast feedback (lint → test → build) | 🟠 High | ☐ |
| **Pipeline** | Pipeline completes in < 10 minutes | 🟠 High | ☐ |
| **Pipeline** | Fail-fast: cheap checks run first | 🟠 High | ☐ |
| **Branching** | Trunk-based development with short-lived branches | 🟠 High | ☐ |
| **Branching** | Branch protection with required status checks | 🔴 Critical | ☐ |
| **Branching** | Required PR reviews before merge | 🔴 Critical | ☐ |
| **Build** | Dependency caching enabled | 🟠 High | ☐ |
| **Build** | Builds are idempotent and reproducible | 🔴 Critical | ☐ |
| **Build** | Lockfiles committed and enforced (`npm ci`) | 🔴 Critical | ☐ |
| **Build** | Parallel stages where possible | 🟡 Medium | ☐ |
| **Artifacts** | Build once, deploy to all environments | 🔴 Critical | ☐ |
| **Artifacts** | Immutable artifacts with Git SHA tags | 🔴 Critical | ☐ |
| **Artifacts** | Semantic versioning for releases | 🟠 High | ☐ |
| **Artifacts** | Never use `latest` tag in production | 🔴 Critical | ☐ |
| **Secrets** | No secrets in code, config files, or logs | 🔴 Critical | ☐ |
| **Secrets** | Secrets from vault / secrets manager | 🔴 Critical | ☐ |
| **Secrets** | Least-privilege permissions per job | 🟠 High | ☐ |
| **Secrets** | Automated secret rotation | 🟡 Medium | ☐ |
| **Testing** | Test pyramid: unit > integration > E2E | 🟠 High | ☐ |
| **Testing** | Coverage threshold enforced (≥ 80%) | 🟠 High | ☐ |
| **Testing** | Security scanning in CI (SAST, dependency audit) | 🔴 Critical | ☐ |
| **Testing** | Flaky tests tracked and quarantined | 🟡 Medium | ☐ |
| **Deploy** | Zero-downtime deployment strategy | 🔴 Critical | ☐ |
| **Deploy** | Automated rollback on failure | 🟠 High | ☐ |
| **Deploy** | Environment parity (same image across envs) | 🔴 Critical | ☐ |
| **Deploy** | Production deploys require approval gate | 🟠 High | ☐ |
| **Observability** | Pipeline duration tracked | 🟡 Medium | ☐ |
| **Observability** | DORA metrics measured | 🟡 Medium | ☐ |
| **Observability** | Deploy notifications to team chat | 🟠 High | ☐ |
| **Observability** | Failed deploys alert on-call | 🔴 Critical | ☐ |
| **Docs** | Pipeline documented (stages, secrets, troubleshooting) | 🟠 High | ☐ |
| **Docs** | Rollback runbook exists | 🔴 Critical | ☐ |

---

## Next Steps

Continue your CI/CD learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | CI/CD Fundamentals | Pipeline anatomy, DevOps culture, branching strategies, artifact management |
| [01-CONTINUOUS-INTEGRATION.md](01-CONTINUOUS-INTEGRATION.md) | Continuous Integration | Automated builds, test automation, merge strategies |
| [02-CONTINUOUS-DELIVERY.md](02-CONTINUOUS-DELIVERY.md) | Continuous Delivery | Deployment pipelines, release strategies, environment promotion |
| [03-GITHUB-ACTIONS.md](03-GITHUB-ACTIONS.md) | GitHub Actions | Workflows, actions marketplace, reusable workflows |
| [06-GITOPS.md](06-GITOPS.md) | GitOps | Git as single source of truth for infrastructure and deployments |
| [07-TESTING-IN-PIPELINES.md](07-TESTING-IN-PIPELINES.md) | Testing in Pipelines | Test automation strategies, coverage, and quality gates |
| [08-SECURITY-IN-PIPELINES.md](08-SECURITY-IN-PIPELINES.md) | Security in Pipelines | SAST, DAST, secret scanning, supply chain security |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial CI/CD Best Practices documentation |
