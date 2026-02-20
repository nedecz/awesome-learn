# CI/CD Fundamentals

## Table of Contents

1. [Overview](#overview)
2. [What is CI/CD](#what-is-cicd)
3. [DevOps Culture and CI/CD](#devops-culture-and-cicd)
4. [Pipeline Anatomy](#pipeline-anatomy)
5. [Pipeline Execution Models](#pipeline-execution-models)
6. [Build, Test, Deploy Lifecycle](#build-test-deploy-lifecycle)
7. [Environments](#environments)
8. [Branching Strategies and Their Impact on CI/CD](#branching-strategies-and-their-impact-on-cicd)
9. [Artifact Management](#artifact-management)
10. [Feedback Loops and Notifications](#feedback-loops-and-notifications)
11. [Prerequisites](#prerequisites)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

This documentation provides a comprehensive introduction to Continuous Integration, Continuous Delivery, and Continuous Deployment (CI/CD). It covers the foundational concepts, pipeline design, DevOps culture, branching strategies, artifact management, and the practical knowledge needed to build, test, and deploy software through automated pipelines.

### Target Audience

- **Developers** writing code that flows through CI/CD pipelines and authoring pipeline definitions alongside application code
- **DevOps Engineers** designing, building, and maintaining CI/CD pipelines across multiple environments and platforms
- **Site Reliability Engineers (SREs)** ensuring deployment reliability, rollback mechanisms, and observability within pipelines
- **Architects** evaluating CI/CD strategies, tool selection, and pipeline architectures for teams and organizations

### Scope

- What CI/CD is and how Continuous Integration, Continuous Delivery, and Continuous Deployment differ
- DevOps culture and how CI/CD enables collaboration between development and operations
- Pipeline anatomy: stages, jobs, steps, triggers, and artifacts
- Pipeline execution models: declarative vs imperative, YAML-based vs UI-based
- The build, test, deploy lifecycle and how each phase contributes to software quality
- Environment promotion strategies from development through production
- Branching strategies and their direct impact on pipeline design
- Artifact management for container images, packages, and binaries
- Feedback loops, notifications, and quality gates

---

## What is CI/CD

**CI/CD** is a set of practices that automate the process of integrating code changes, validating them through testing, and delivering or deploying them to target environments. The acronym encompasses three distinct but related practices: **Continuous Integration (CI)**, **Continuous Delivery (CD)**, and **Continuous Deployment (CD)**.

### The Three Pillars

```
    Continuous Integration          Continuous Delivery          Continuous Deployment
  ┌───────────────────────┐    ┌───────────────────────┐    ┌───────────────────────┐
  │                       │    │                       │    │                       │
  │  Developers merge     │    │  Code is always in a  │    │  Every change that    │
  │  code frequently      │    │  deployable state     │    │  passes the pipeline  │
  │  into a shared branch │───►│                       │───►│  is automatically     │
  │                       │    │  Deployment to prod   │    │  deployed to          │
  │  Automated build and  │    │  requires manual      │    │  production           │
  │  test on every change │    │  approval / trigger   │    │                       │
  │                       │    │                       │    │  No manual gates      │
  └───────────────────────┘    └───────────────────────┘    └───────────────────────┘

  "Did I break anything?"      "Can we release this?"       "Ship it — automatically"
```

### Continuous Integration (CI)

Continuous Integration is the practice of frequently merging code changes into a shared repository, where each merge triggers an automated build and test cycle. The goal is to detect integration problems early, when they are small and easy to fix.

**Core principles of CI:**

- Developers commit to the mainline (or merge via pull requests) at least once per day
- Every commit triggers an automated build and test run
- Build and test failures are treated as the highest priority — fix them before writing new code
- The mainline is always in a working state

```yaml
# Example: A basic CI workflow (GitHub Actions)
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build
```

### Continuous Delivery (CD)

Continuous Delivery extends CI by ensuring that the software is always in a releasable state. After every change passes the automated pipeline, it is ready to be deployed to production — but the actual deployment requires a manual trigger or approval.

**Core principles of Continuous Delivery:**

- Every change that passes automated testing is a release candidate
- Deployment to any environment is a push-button decision
- The pipeline includes automated acceptance tests, security scans, and performance checks
- Rollback is always possible and well-tested

### Continuous Deployment (CD)

Continuous Deployment takes Continuous Delivery one step further: every change that passes all stages of the pipeline is automatically deployed to production without human intervention. This requires extremely high confidence in the automated test suite and monitoring.

**Core principles of Continuous Deployment:**

- No manual approval gates between the pipeline and production
- Feature flags decouple deployment from feature release
- Comprehensive monitoring and automated rollback are essential
- The team has high confidence in test coverage and observability

### CI vs Continuous Delivery vs Continuous Deployment

| Aspect | Continuous Integration | Continuous Delivery | Continuous Deployment |
|---|---|---|---|
| **Focus** | Merging and validating code | Keeping code releasable | Automatically releasing code |
| **Automation** | Build + test on every commit | Build + test + staging deploy | Build + test + production deploy |
| **Production deploy** | Not included | Manual trigger / approval | Fully automated |
| **Risk** | Low — catch bugs early | Low — always releasable | Very low — fast feedback from prod |
| **Requires** | Automated tests, CI server | Automated tests + deployment pipeline | Automated tests + monitoring + feature flags |
| **Feedback speed** | Minutes (build + test) | Minutes to hours (pipeline) | Minutes (code to production) |
| **Team maturity** | Entry level | Intermediate | Advanced |

---

## DevOps Culture and CI/CD

**DevOps** is a cultural and technical movement that bridges the gap between software development (Dev) and IT operations (Ops). CI/CD is the technical backbone that makes DevOps practices actionable — it automates the handoffs that traditionally caused friction between teams.

### The Wall of Confusion

Before DevOps, development and operations teams worked in silos with conflicting goals:

```
  Development Team                    Operations Team
  ────────────────                    ────────────────
  Goal: Ship features fast            Goal: Keep systems stable
  Metric: Velocity                    Metric: Uptime
  Risk: Change is good               Risk: Change is dangerous

               ┌──────────────────┐
               │                  │
               │  "The Wall of    │
  "Throw it ──►│   Confusion"     │──► "We didn't build it,
   over the    │                  │     we can't fix it"
   wall"       │  Manual handoffs │
               │  Blame culture   │
               │  Slow releases   │
               │  Painful deploys │
               └──────────────────┘
```

### How CI/CD Breaks Down the Wall

CI/CD replaces manual, error-prone handoffs with automated, repeatable pipelines that both teams own:

| Traditional (Pre-DevOps) | With CI/CD (DevOps) |
|---|---|
| Manual builds on a developer's machine | Automated builds in a clean, reproducible environment |
| "Works on my machine" deployments | Identical artifacts deployed to every environment |
| Weekly or monthly release cycles | Multiple deployments per day |
| Manual testing before release | Automated test suites run on every commit |
| Operations receive a tarball and a prayer | Infrastructure as code, pipeline as code, everything versioned |
| Post-mortem blame game | Shared ownership, blameless retrospectives |
| Change Advisory Board bottleneck | Automated quality gates with policy as code |

### The DORA Metrics

The **DevOps Research and Assessment (DORA)** team identified four key metrics that measure software delivery performance. CI/CD directly improves all four:

| Metric | Definition | How CI/CD Helps |
|---|---|---|
| **Deployment Frequency** | How often code is deployed to production | Automated pipelines reduce friction; deploying becomes routine |
| **Lead Time for Changes** | Time from commit to production | Automation eliminates manual steps and waiting |
| **Change Failure Rate** | Percentage of deployments causing failures | Automated testing catches defects before production |
| **Mean Time to Restore (MTTR)** | Time to recover from a production failure | Automated rollback, fast re-deploy, and observability |

```
  DORA Performance Levels and CI/CD Maturity

  Elite          ──►  Multiple deploys per day, < 1 hour lead time,
                      < 5% failure rate, < 1 hour MTTR
                      CI/CD: Continuous Deployment, feature flags,
                      automated rollback, full observability

  High           ──►  Between once per day and once per week,
                      < 1 day lead time, 0-15% failure rate
                      CI/CD: Continuous Delivery, automated staging,
                      manual production approval

  Medium         ──►  Between once per week and once per month,
                      1 day - 1 week lead time, 0-15% failure rate
                      CI/CD: CI with automated tests, manual deployment

  Low            ──►  Between once per month and once every 6 months,
                      1 month - 6 months lead time, 16-30% failure rate
                      CI/CD: Manual builds, little to no automation
```

### Shared Ownership

In a DevOps culture, the CI/CD pipeline is not owned by a single team — it is a shared responsibility:

- **Developers** write tests, maintain pipeline definitions, and respond to build failures
- **Operations** define infrastructure as code, manage runners/agents, and ensure environment parity
- **Security** integrates scanning tools into the pipeline (shift-left security)
- **QA** contributes automated test suites and defines quality gates
- **Everyone** monitors pipeline health, deployment success, and production stability

---

## Pipeline Anatomy

A **CI/CD pipeline** is an automated workflow that takes code from a developer's commit through build, test, and deployment stages to a target environment. Pipelines are composed of **triggers**, **stages**, **jobs**, **steps**, and **artifacts**.

### Pipeline Components

```
                            CI/CD Pipeline Anatomy

  Trigger                   Stages                              Output
  ───────                   ──────                              ──────
  ┌─────────┐    ┌────────────────────────────────────────┐    ┌──────────┐
  │  push   │    │  ┌───────┐  ┌───────┐  ┌───────────┐  │    │ Artifact │
  │  PR     │───►│  │ Build │─►│ Test  │─►│  Deploy   │  │───►│ Registry │
  │  tag    │    │  └───┬───┘  └───┬───┘  └─────┬─────┘  │    │          │
  │  cron   │    │      │          │            │         │    │ Reports  │
  │  manual │    │   ┌──┴──┐   ┌──┴──┐    ┌────┴────┐   │    │          │
  └─────────┘    │   │Job 1│   │Job 1│    │ Job 1   │   │    │ Deployed │
                 │   │Job 2│   │Job 2│    │ (dev)   │   │    │ App      │
                 │   └──┬──┘   │Job 3│    │ Job 2   │   │    └──────────┘
                 │      │      └──┬──┘    │ (stg)   │   │
                 │   ┌──┴──┐     │       │ Job 3   │   │
                 │   │Steps│   ┌──┴──┐    │ (prod)  │   │
                 │   │ · · │   │Steps│    └─────────┘   │
                 │   └─────┘   │ · · │                  │
                 │             └─────┘                  │
                 └────────────────────────────────────────┘
```

### Triggers

Triggers are events that start a pipeline run. Most CI/CD platforms support a wide range of trigger types:

| Trigger Type | Description | Example |
|---|---|---|
| **Push** | Code pushed to a branch | Push to `main` triggers a production pipeline |
| **Pull Request** | PR opened, updated, or merged | Run tests and linting on every PR |
| **Tag** | A Git tag is created | Tag `v1.2.0` triggers a release pipeline |
| **Schedule (Cron)** | Time-based trigger | Nightly builds, weekly security scans |
| **Manual** | Human-initiated pipeline run | Deploy to production with a button click |
| **Webhook** | External event via HTTP | Trigger pipeline when a dependency is updated |
| **Pipeline** | Completion of another pipeline | Deploy pipeline starts after build pipeline succeeds |

```yaml
# Example: Multiple triggers (GitHub Actions)
on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'  # Every Monday at 2:00 AM UTC
  workflow_dispatch:       # Manual trigger
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
```

### Stages

A **stage** is a logical grouping of related jobs that represents a phase of the pipeline (e.g., Build, Test, Deploy). Stages typically run sequentially — the next stage starts only after the previous stage completes successfully.

### Jobs

A **job** is a unit of work that runs on a single runner or agent. Jobs within a stage can run in parallel (to speed up the pipeline) or sequentially (when dependencies exist between them).

### Steps

A **step** is an individual command, script, or action within a job. Steps within a job run sequentially on the same runner, sharing the same filesystem and environment.

```yaml
# Example: Stages, jobs, and steps (GitHub Actions)
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4          # Step 1: checkout code
      - run: npm ci                        # Step 2: install dependencies
      - run: npm run build                 # Step 3: build the application
      - uses: actions/upload-artifact@v4   # Step 4: upload build artifact
        with:
          name: build-output
          path: dist/

  test:
    needs: build                           # Sequential dependency on build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 22]         # Parallel: test across Node versions
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm test

  deploy:
    needs: test                            # Sequential dependency on test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'    # Only deploy from main
    steps:
      - uses: actions/checkout@v4
      - run: ./scripts/deploy.sh
```

### Artifacts

An **artifact** is any output produced by a pipeline stage that is consumed by a later stage or stored for future reference. Common artifacts include compiled binaries, container images, test reports, and coverage data.

---

## Pipeline Execution Models

CI/CD pipelines can be defined and executed in different ways. The two primary dimensions are **declarative vs imperative** and **YAML-based vs UI-based**.

### Declarative vs Imperative

| Aspect | Declarative | Imperative |
|---|---|---|
| **Approach** | Describe *what* you want the pipeline to do | Describe *how* the pipeline should execute step by step |
| **Abstraction** | Higher — the platform handles execution details | Lower — you control the execution flow |
| **Readability** | Easier to read and understand at a glance | More flexible but harder to follow |
| **Error handling** | Platform manages retries, cleanup, and ordering | You must code retry logic, conditionals, and error handlers |
| **Examples** | GitHub Actions YAML, Azure Pipelines YAML, GitLab CI | Jenkins scripted pipeline, shell scripts |
| **Best for** | Standard build-test-deploy workflows | Complex pipelines with conditional logic and dynamic behavior |

```yaml
# Declarative: GitHub Actions
# "What" — declare the desired stages and let the platform handle execution
name: Build and Deploy
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

```groovy
// Imperative: Jenkins Scripted Pipeline
// "How" — programmatic control with Groovy scripting
node {
    try {
        stage('Checkout') {
            checkout scm
        }
        stage('Build') {
            sh 'npm ci'
            sh 'npm run build'
        }
        stage('Deploy') {
            if (env.BRANCH_NAME == 'main') {
                sh './deploy.sh'
            } else {
                echo "Skipping deploy for branch ${env.BRANCH_NAME}"
            }
        }
    } catch (Exception e) {
        currentBuild.result = 'FAILURE'
        // Send notification on failure
        slackSend channel: '#builds', message: "Build failed: ${e.message}"
        throw e
    }
}
```

### YAML-Based vs UI-Based

| Aspect | YAML-Based (Pipeline as Code) | UI-Based (Click-to-Configure) |
|---|---|---|
| **Definition** | Pipeline defined in a YAML file in the repository | Pipeline configured through a web interface |
| **Version control** | Yes — lives alongside application code in Git | No — configuration stored in the CI/CD platform |
| **Code review** | Pipeline changes go through pull requests | No review process for pipeline changes |
| **Reproducibility** | Fully reproducible from any Git commit | Depends on platform state; hard to reproduce |
| **Collaboration** | Teams collaborate via Git workflows | Single person makes changes in the UI |
| **Portability** | Can migrate between platforms (with translation) | Locked to the specific CI/CD platform |
| **Learning curve** | Requires learning YAML syntax and platform DSL | Lower barrier to entry; visual and interactive |
| **Examples** | GitHub Actions, GitLab CI, Azure Pipelines, CircleCI | Jenkins (classic), TeamCity, Bamboo (classic) |

**Industry trend:** YAML-based pipeline-as-code is the dominant approach in modern CI/CD. Defining pipelines in version-controlled files ensures reproducibility, enables code review for pipeline changes, and treats infrastructure and delivery with the same rigor as application code.

### Platform Comparison

```
  Pipeline Definition by Platform

  ┌─────────────────────────────────────────────────────────────────┐
  │  Platform              Format          File                     │
  ├─────────────────────────────────────────────────────────────────┤
  │  GitHub Actions        YAML            .github/workflows/*.yml  │
  │  GitLab CI             YAML            .gitlab-ci.yml           │
  │  Azure Pipelines       YAML            azure-pipelines.yml      │
  │  CircleCI              YAML            .circleci/config.yml     │
  │  Jenkins (modern)      Groovy          Jenkinsfile              │
  │  Tekton                YAML (K8s CRD)  *.yaml                   │
  │  Argo Workflows        YAML (K8s CRD)  *.yaml                   │
  └─────────────────────────────────────────────────────────────────┘
```

---

## Build, Test, Deploy Lifecycle

The build-test-deploy lifecycle is the core sequence that every CI/CD pipeline follows. Each phase serves a distinct purpose and contributes to the overall confidence that a change is safe to release.

```
                    Build, Test, Deploy Lifecycle

  ┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌───────────┐
  │  Source   │    │    Build     │    │     Test      │    │  Deploy   │
  │          │    │              │    │              │    │           │
  │ Checkout │───►│ Compile      │───►│ Unit tests   │───►│ Dev       │
  │ Fetch    │    │ Resolve deps │    │ Lint / SAST  │    │ Staging   │
  │ deps     │    │ Package      │    │ Integration  │    │ Production│
  │          │    │ Containerize │    │ E2E / Smoke  │    │           │
  └──────────┘    └──────┬───────┘    └──────┬───────┘    └─────┬─────┘
                         │                   │                  │
                         ▼                   ▼                  ▼
                   ┌───────────┐       ┌───────────┐      ┌───────────┐
                   │ Artifact  │       │ Reports   │      │ Running   │
                   │ (binary,  │       │ (coverage,│      │ Service   │
                   │  image)   │       │  results) │      │           │
                   └───────────┘       └───────────┘      └───────────┘
```

### Source Phase

The source phase is the entry point of every pipeline. It checks out the code, restores dependencies, and prepares the workspace for subsequent phases.

```yaml
# Source phase example
steps:
  - uses: actions/checkout@v4
    with:
      fetch-depth: 0  # Full history for versioning and changelog

  - name: Cache dependencies
    uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-npm-

  - name: Install dependencies
    run: npm ci
```

### Build Phase

The build phase compiles source code, resolves dependencies, and produces a deployable artifact. For interpreted languages, this may involve bundling, transpilation, or container image creation.

```yaml
# Build phase example — multi-language
steps:
  # Go application
  - name: Build Go binary
    run: |
      CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
        go build -ldflags="-s -w" -o ./bin/app ./cmd/server

  # Container image
  - name: Build container image
    run: |
      docker build \
        --tag my-app:${{ github.sha }} \
        --tag my-app:latest \
        --build-arg VERSION=${{ github.sha }} \
        .
```

### Test Phase

The test phase validates the change through multiple layers of testing, each providing a different level of confidence:

| Test Type | Scope | Speed | Confidence | When to Run |
|---|---|---|---|---|
| **Unit tests** | Individual functions and classes | Very fast (seconds) | Low-medium (isolated logic) | Every commit |
| **Linting / SAST** | Code style and static analysis | Fast (seconds) | Medium (catches common errors) | Every commit |
| **Integration tests** | Component interactions, APIs, databases | Medium (minutes) | Medium-high | Every commit or PR |
| **End-to-end tests** | Full user workflows through the UI | Slow (minutes to hours) | High (full system validation) | PR merge, pre-deploy |
| **Performance tests** | Load, stress, latency | Slow (minutes to hours) | High (non-functional requirements) | Nightly or pre-release |
| **Security scans** | Dependency vulnerabilities, secrets | Medium (minutes) | High (compliance, risk) | Every commit |

```yaml
# Test phase example — layered testing
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test -- --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/

  lint-and-sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint
      - name: Run SAST scan
        uses: github/codeql-action/analyze@v3

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:integration
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/testdb
```

### Deploy Phase

The deploy phase pushes the validated artifact to a target environment. Deployments should be automated, repeatable, and reversible.

```yaml
# Deploy phase example — environment promotion
deploy-staging:
  needs: [unit-tests, integration-tests]
  runs-on: ubuntu-latest
  environment:
    name: staging
    url: https://staging.example.com
  steps:
    - uses: actions/checkout@v4
    - name: Deploy to staging
      run: |
        kubectl set image deployment/my-app \
          my-app=registry.example.com/my-app:${{ github.sha }} \
          --namespace staging

deploy-production:
  needs: deploy-staging
  runs-on: ubuntu-latest
  environment:
    name: production
    url: https://www.example.com
  steps:
    - uses: actions/checkout@v4
    - name: Deploy to production
      run: |
        kubectl set image deployment/my-app \
          my-app=registry.example.com/my-app:${{ github.sha }} \
          --namespace production
```

---

## Environments

An **environment** is a distinct deployment target where the application runs. Environments provide isolation between stages of the delivery process and allow teams to validate changes progressively before they reach end users.

### Environment Promotion Model

```
                      Environment Promotion

  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌──────────────┐
  │    Dev       │     │   Staging   │     │  Pre-Prod   │     │  Production  │
  │             │     │             │     │  (Optional) │     │              │
  │ Auto-deploy │────►│ Auto-deploy │────►│ Manual gate │────►│ Manual gate  │
  │ on merge    │     │ after CI    │     │ + load test │     │ + approval   │
  │             │     │ passes      │     │             │     │              │
  │ Developers  │     │ QA / Dev    │     │ SRE / QA    │     │ End users    │
  │ test here   │     │ validate    │     │ validate    │     │              │
  └─────────────┘     └─────────────┘     └─────────────┘     └──────────────┘

  Low confidence ◄─────────────────────────────────────────► High confidence
  Fast feedback                                              Slower feedback
  Frequent deploys                                           Controlled releases
```

### Environment Characteristics

| Environment | Purpose | Deploy Trigger | Data | Stability | Access |
|---|---|---|---|---|---|
| **Development** | Developer testing and integration | Automatic on merge to main | Synthetic/seed data | Low — frequent changes | Developers |
| **Staging** | Pre-production validation | Automatic after CI passes | Production-like sample data | Medium — mirrors prod config | Dev + QA |
| **Pre-Production** | Final validation before release | Manual or scheduled | Anonymized production data | High — exact prod replica | SRE + QA |
| **Production** | Live user traffic | Manual approval or auto-deploy | Real user data | Highest — change-controlled | End users |

### Environment Parity

Environment parity means keeping all environments as similar as possible to production. The closer your staging environment matches production, the fewer surprises you encounter during deployment.

**What should be identical across environments:**

- Operating system and runtime versions
- Infrastructure configuration (via Infrastructure as Code)
- Application dependencies and versions
- Network topology and security policies
- Monitoring and logging configuration

**What should differ across environments:**

- Credentials and secrets (unique per environment)
- Scale (fewer replicas in dev/staging)
- Data sources (synthetic in dev, anonymized in staging)
- External service endpoints (sandbox APIs in non-prod)

```yaml
# Environment configuration example (GitHub Actions)
jobs:
  deploy:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, production]
    environment:
      name: ${{ matrix.environment }}
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to ${{ matrix.environment }}
        run: |
          helm upgrade --install my-app ./charts/my-app \
            --namespace ${{ matrix.environment }} \
            --set image.tag=${{ github.sha }} \
            --values ./charts/my-app/values-${{ matrix.environment }}.yaml
```

---

## Branching Strategies and Their Impact on CI/CD

The branching strategy a team adopts directly shapes how CI/CD pipelines are designed, triggered, and optimized. Different strategies trade off between isolation, complexity, and speed of delivery.

### Trunk-Based Development

In trunk-based development, all developers commit directly to the main branch (the "trunk") or merge short-lived feature branches (lasting hours to a few days). This strategy is optimized for Continuous Deployment.

```
  Trunk-Based Development

  main ─────●─────●─────●─────●─────●─────●──────► (always deployable)
              \   /       \   /       \   /
  feature/a    ●─●         │ │         │ │
  (1-2 days)              ●─●         │ │
  feature/b           (1-2 days)      │ │
                                      ●─●
  feature/c                       (1-2 days)

  Pipeline: Runs on every push to main and every PR
  Deploy:   Automatic to production from main
  Flags:    Feature flags control visibility of incomplete work
```

| Advantage | Description |
|---|---|
| **Fast feedback** | Small changes are tested and integrated quickly |
| **Fewer merge conflicts** | Short-lived branches minimize divergence |
| **Always deployable** | Main branch is always in a releasable state |
| **Simple pipeline** | One pipeline for main; no complex branch management |

### Feature Branch Workflow

Each feature or bug fix is developed in a dedicated branch that is merged to main via a pull request. This is the most common workflow for teams using GitHub or GitLab.

```
  Feature Branch Workflow

  main ─────●───────────●───────────●───────────●──────► (stable)
              \         / \         / \         /
  feature/    ●───●───●    │         │  │         │
  login       (PR + CI)    │         │  │         │
                           │         │  │         │
  feature/                 ●───●───● │  │         │
  dashboard                (PR + CI) │  │         │
                                     │  │         │
  bugfix/                            ●──●         │
  header                          (PR + CI)       │
                                                  │
  feature/                                        ●───●───●
  payments                                        (PR + CI)

  Pipeline: Runs on every PR (build + test), on merge to main (deploy)
  Deploy:   Automatic or manual from main after PR merge
```

### Release Branch Workflow

Release branches are cut from main when a release is being prepared. Bug fixes are applied to the release branch and merged back. This strategy is common in teams that ship versioned software.

```
  Release Branch Workflow

  main ─────●─────●─────●─────●─────●─────●─────●──────►
              \                       \
  release/     ●─────●─────●           ●─────●
  1.0         (stabilize, (release      (release
               hotfix)     v1.0)        v2.0)
                    \
  hotfix/            ●
  1.0.1          (cherry-pick
                  to release
                  and main)

  Pipeline: Runs on main (CI), on release branches (CI + deploy to staging/prod)
  Deploy:   Staging from release branch, production after approval
  Tags:     Release tags trigger production deployments
```

### Strategy Comparison

| Strategy | Branch Lifetime | Merge Frequency | Pipeline Complexity | Best For |
|---|---|---|---|---|
| **Trunk-based** | Hours to 1-2 days | Multiple times daily | Low | Continuous Deployment, mature teams |
| **Feature branch** | Days to 1-2 weeks | Daily to weekly | Medium | Most teams, PR-based review |
| **Release branch** | Weeks to months | At release milestones | High | Versioned products, mobile apps, enterprise |
| **GitFlow** | Variable | At release milestones | Very high | Legacy products with strict release cycles |

---

## Artifact Management

An **artifact** is any output produced by a CI/CD pipeline — compiled binaries, container images, test reports, documentation bundles, or infrastructure packages. Proper artifact management ensures traceability, reproducibility, and security across the delivery pipeline.

### Types of Artifacts

| Artifact Type | Examples | Storage | Lifecycle |
|---|---|---|---|
| **Container images** | Docker images, OCI images | Container registry (Docker Hub, ECR, ACR, GHCR) | Tagged by version and SHA; retained per policy |
| **Packages** | npm packages, NuGet packages, Python wheels, Maven JARs | Package registry (npm, NuGet, PyPI, Maven Central) | Versioned per semver; immutable once published |
| **Binaries** | Compiled executables, .deb/.rpm packages | Artifact store (Artifactory, Nexus, S3) | Versioned; retained per retention policy |
| **Test reports** | JUnit XML, coverage HTML, Allure reports | Pipeline artifact storage | Retained for audit; pruned after a retention period |
| **Infrastructure** | Terraform plans, Helm charts, CloudFormation templates | Chart/module registry, S3, OCI registry | Versioned alongside application releases |

### Container Image Tagging Strategy

A well-defined tagging strategy ensures you can always trace a running container back to the exact code that produced it:

```bash
# Tag by Git SHA (immutable, traceable)
docker build -t registry.example.com/my-app:abc123f .

# Tag by semantic version (human-readable)
docker build -t registry.example.com/my-app:1.2.3 .

# Tag by branch (mutable, useful for dev/staging)
docker build -t registry.example.com/my-app:main .

# Combined strategy (recommended)
IMAGE="registry.example.com/my-app"
SHA=$(git rev-parse --short HEAD)
VERSION=$(git describe --tags --always)

docker build \
  -t "${IMAGE}:${SHA}" \
  -t "${IMAGE}:${VERSION}" \
  -t "${IMAGE}:latest" \
  .

docker push "${IMAGE}:${SHA}"
docker push "${IMAGE}:${VERSION}"
docker push "${IMAGE}:latest"
```

### Artifact Flow Through the Pipeline

```
  Artifact Flow

  ┌──────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │  Build   │    │   Publish    │    │   Deploy     │    │   Verify     │
  │          │    │              │    │              │    │              │
  │ Compile  │───►│ Push image   │───►│ Pull image   │───►│ Smoke tests  │
  │ Package  │    │ to registry  │    │ from registry│    │ Health check │
  │ Sign     │    │ Upload pkg   │    │ Deploy to env│    │ Monitoring   │
  └──────────┘    └──────┬───────┘    └──────────────┘    └──────────────┘
                         │
                    ┌────┴────┐
                    │Registry │
                    │         │
                    │ Stored  │
                    │ Scanned │
                    │ Signed  │
                    └─────────┘
```

### Artifact Security

Securing the artifact pipeline is critical for supply chain integrity:

- **Sign artifacts** — use Sigstore/Cosign to cryptographically sign container images
- **Scan for vulnerabilities** — run image scanners (Trivy, Grype, Snyk) in the pipeline
- **Use digest references** — pin images by SHA256 digest, not mutable tags
- **Enforce provenance** — generate and verify SLSA provenance attestations
- **Retention policies** — automatically delete old, unused artifacts to reduce attack surface

```yaml
# Artifact security example — sign and scan a container image
steps:
  - name: Build image
    run: docker build -t my-app:${{ github.sha }} .

  - name: Scan image for vulnerabilities
    uses: aquasecurity/trivy-action@master
    with:
      image-ref: my-app:${{ github.sha }}
      format: 'sarif'
      output: 'trivy-results.sarif'
      severity: 'CRITICAL,HIGH'

  - name: Sign image with Cosign
    run: |
      cosign sign --yes \
        registry.example.com/my-app:${{ github.sha }}
```

---

## Feedback Loops and Notifications

Fast, actionable feedback is the defining characteristic of an effective CI/CD pipeline. Every pipeline run should inform developers whether their change is safe to ship — and if not, exactly what went wrong.

### The Feedback Loop

```
                         CI/CD Feedback Loop

  ┌──────────┐     ┌──────────────┐     ┌───────────────┐
  │Developer │     │   Pipeline   │     │  Notification  │
  │          │     │              │     │               │
  │  Commit  │────►│  Build       │────►│  ✓ Success    │──┐
  │  Push    │     │  Test        │     │  ✗ Failure    │  │
  │  PR      │     │  Scan        │     │  ⚠ Warning    │  │
  │          │     │  Deploy      │     │               │  │
  └──────────┘     └──────────────┘     └───────────────┘  │
       ▲                                                    │
       │                                                    │
       └────────────────────────────────────────────────────┘
                    Fix, iterate, improve
```

### Notification Channels

| Channel | Best For | Latency | Actionability |
|---|---|---|---|
| **PR status checks** | Blocking merges on failure | Real-time | High — visible in the PR workflow |
| **Email** | Detailed reports, audit trail | Minutes | Medium — easy to miss |
| **Slack / Teams** | Team-wide awareness | Real-time | Medium — can cause notification fatigue |
| **Dashboard** | Historical trends, team metrics | On-demand | High — visual and contextual |
| **PagerDuty / OpsGenie** | Production deployment failures | Real-time | Very high — alerts on-call engineers |
| **IDE integration** | Inline test failures, lint errors | Real-time | Very high — fix without leaving the editor |

### Quality Gates

A **quality gate** is a checkpoint in the pipeline that blocks progression if a condition is not met. Quality gates enforce standards automatically and remove the need for manual review of routine checks.

| Quality Gate | Condition | Blocks |
|---|---|---|
| **Build success** | Code compiles without errors | All subsequent stages |
| **Unit test pass rate** | 100% of tests pass | Merge to main |
| **Code coverage** | Coverage ≥ 80% (configurable threshold) | Merge to main |
| **Linting** | Zero lint errors | Merge to main |
| **Security scan** | No critical or high vulnerabilities | Deploy to staging / production |
| **Performance** | Response time < 200ms (p95) | Deploy to production |
| **Manual approval** | Human sign-off | Deploy to production |

```yaml
# Quality gates example — branch protection + required checks
# Configure in GitHub: Settings → Branches → Branch protection rules
#
# Required status checks before merging:
#   ✓ build
#   ✓ unit-tests
#   ✓ lint
#   ✓ security-scan
#
# Additional protection:
#   ✓ Require pull request reviews (1 approval)
#   ✓ Require conversation resolution
#   ✓ Require signed commits

# Pipeline definition with gates
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint

  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test -- --coverage
      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | \
            jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% is below 80% threshold"
            exit 1
          fi

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run security scan
        uses: github/codeql-action/analyze@v3

  deploy-staging:
    needs: [lint, unit-tests, security-scan]  # All gates must pass
    runs-on: ubuntu-latest
    environment:
      name: staging
    steps:
      - run: echo "Deploying to staging..."

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production  # Requires manual approval in GitHub
    steps:
      - run: echo "Deploying to production..."
```

### Pipeline Metrics to Track

Monitoring pipeline health is essential for maintaining developer productivity:

| Metric | What It Measures | Target |
|---|---|---|
| **Pipeline duration** | Total time from trigger to completion | < 10 minutes for CI, < 30 minutes for full CD |
| **Build success rate** | Percentage of pipeline runs that succeed | > 95% |
| **Flaky test rate** | Percentage of tests that pass/fail non-deterministically | < 1% |
| **Queue time** | Time a job waits for a runner before starting | < 1 minute |
| **Mean time to fix (MTTF)** | Time from pipeline failure to passing again | < 1 hour |
| **Deploy frequency** | How often the team deploys to production | Multiple times per day (elite) |

---

## Prerequisites

### Required Knowledge

Before working through the CI/CD series, you should be familiar with:

| Topic | Why It Matters |
|---|---|
| **Version control (Git)** | CI/CD pipelines are triggered by Git events; branching strategies directly shape pipeline design |
| **Linux command line** | Pipeline steps execute shell commands; debugging requires comfort with the terminal |
| **A programming language** | Building, testing, and packaging an application requires understanding its toolchain |
| **Basic networking** | Deployments involve HTTP endpoints, DNS, load balancers, and firewall rules |
| **YAML syntax** | Most modern CI/CD platforms use YAML for pipeline definitions |
| **Containers (Docker)** | Many pipelines build and deploy container images; understanding Docker basics is highly recommended |

### Required Tools

Install the following tools to follow hands-on examples:

```bash
# Git (version control — required for all CI/CD workflows)
# macOS
brew install git
# Ubuntu/Debian
sudo apt-get install -y git

# GitHub CLI (interact with GitHub Actions, PRs, and repos)
# macOS
brew install gh
# Ubuntu/Debian
sudo apt-get install -y gh
# Authenticate
gh auth login

# Docker (build and push container images in pipelines)
# macOS
brew install --cask docker
# Ubuntu/Debian
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# kubectl (deploy to Kubernetes from pipelines)
# macOS
brew install kubectl
# Ubuntu/Debian
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Helm (package manager for Kubernetes — used in deploy stages)
# macOS
brew install helm
# Ubuntu/Debian
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# act (run GitHub Actions workflows locally for testing)
# macOS
brew install act
# Linux
curl -s https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# yamllint (lint YAML pipeline definitions)
pip install yamllint

# jq (parse JSON output from CI/CD APIs)
# macOS
brew install jq
# Ubuntu/Debian
sudo apt-get install -y jq

# Verify installations
git --version
gh --version
docker version
kubectl version --client
helm version
act --version
yamllint --version
jq --version
```

---

## Next Steps

Work through the CI/CD series in order, or jump to the topic most relevant to you:

| File | Topic | Description |
|---|---|---|
| [01-CONTINUOUS-INTEGRATION.md](01-CONTINUOUS-INTEGRATION.md) | Continuous Integration | Build automation, testing in CI, artifact management, caching strategies |
| [02-CONTINUOUS-DELIVERY.md](02-CONTINUOUS-DELIVERY.md) | Continuous Delivery | Deployment pipelines, environment promotion, release strategies |
| [03-GITHUB-ACTIONS.md](03-GITHUB-ACTIONS.md) | GitHub Actions | Workflows, actions, runners, reusable workflows, matrix builds |
| [04-AZURE-DEVOPS.md](04-AZURE-DEVOPS.md) | Azure DevOps | Azure Pipelines, YAML pipelines, service connections, variable groups |
| [05-JENKINS.md](05-JENKINS.md) | Jenkins | Jenkinsfile, shared libraries, pipeline as code, distributed builds |
| [06-GITOPS.md](06-GITOPS.md) | GitOps | ArgoCD, Flux, pull-based deployments, declarative configuration |
| [07-TESTING-IN-PIPELINES.md](07-TESTING-IN-PIPELINES.md) | Testing in Pipelines | Unit/integration/E2E tests, test parallelization, quality gates |
| [08-SECURITY-IN-PIPELINES.md](08-SECURITY-IN-PIPELINES.md) | Security in Pipelines | SAST, DAST, dependency scanning, supply chain security |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Pipeline patterns, trunk-based development, feature flags |
| [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md) | Anti-Patterns | Common CI/CD mistakes and how to avoid them |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Structured learning guide with exercises |

### Suggested Learning Path by Role

```
Developer:
  00-OVERVIEW → 01-CONTINUOUS-INTEGRATION → 03-GITHUB-ACTIONS → 07-TESTING-IN-PIPELINES

DevOps / Platform Engineer:
  00-OVERVIEW → 01-CONTINUOUS-INTEGRATION → 02-CONTINUOUS-DELIVERY → 06-GITOPS → 09-BEST-PRACTICES

SRE:
  00-OVERVIEW → 08-SECURITY-IN-PIPELINES → 02-CONTINUOUS-DELIVERY → 06-GITOPS → 10-ANTI-PATTERNS

Architect:
  00-OVERVIEW → 09-BEST-PRACTICES → 06-GITOPS → 08-SECURITY-IN-PIPELINES → 10-ANTI-PATTERNS
```

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial CI/CD fundamentals overview documentation |
