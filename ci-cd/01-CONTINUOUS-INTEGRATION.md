# Continuous Integration

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [What Is Continuous Integration](#what-is-continuous-integration)
   - [History and Origins](#history-and-origins)
   - [Core Principles](#core-principles)
   - [CI vs Continuous Delivery vs Continuous Deployment](#ci-vs-continuous-delivery-vs-continuous-deployment)
3. [The CI Feedback Loop](#the-ci-feedback-loop)
   - [Commit](#commit)
   - [Build](#build)
   - [Test](#test)
   - [Report](#report)
   - [The Fast Feedback Imperative](#the-fast-feedback-imperative)
4. [Build Automation](#build-automation)
   - [Build Systems](#build-systems)
   - [Reproducible Builds](#reproducible-builds)
   - [Build Caching](#build-caching)
   - [Build Automation Example](#build-automation-example)
5. [Source Code Integration](#source-code-integration)
   - [Merge Frequency](#merge-frequency)
   - [Integration Branches](#integration-branches)
   - [Pull Request Validation](#pull-request-validation)
   - [Branch Protection Rules](#branch-protection-rules)
6. [Automated Testing in CI](#automated-testing-in-ci)
   - [Unit Tests](#unit-tests)
   - [Integration Tests](#integration-tests)
   - [Test Coverage](#test-coverage)
   - [Test Reporting](#test-reporting)
   - [Test Parallelism](#test-parallelism)
7. [Artifact Management](#artifact-management)
   - [What Is a Build Artifact](#what-is-a-build-artifact)
   - [Versioning Artifacts](#versioning-artifacts)
   - [Artifact Repositories](#artifact-repositories)
   - [Artifact Repository Comparison](#artifact-repository-comparison)
8. [Code Quality Gates](#code-quality-gates)
   - [Linting](#linting)
   - [Static Analysis](#static-analysis)
   - [Code Coverage Thresholds](#code-coverage-thresholds)
   - [Quality Gate Pipeline Example](#quality-gate-pipeline-example)
9. [CI Server Architecture](#ci-server-architecture)
   - [Agents and Runners](#agents-and-runners)
   - [Build Queues](#build-queues)
   - [Parallelism and Concurrency](#parallelism-and-concurrency)
   - [Self-Hosted vs Cloud-Hosted](#self-hosted-vs-cloud-hosted)
   - [CI Platform Comparison](#ci-platform-comparison)
10. [Monorepo CI Considerations](#monorepo-ci-considerations)
    - [Change Detection](#change-detection)
    - [Selective Builds](#selective-builds)
    - [Monorepo CI Example](#monorepo-ci-example)
11. [Dependency Management in CI](#dependency-management-in-ci)
    - [Lock Files and Determinism](#lock-files-and-determinism)
    - [Dependency Caching](#dependency-caching)
    - [Automated Dependency Updates](#automated-dependency-updates)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **Continuous Integration (CI)** — the practice of automatically building, testing, and validating code changes every time a developer pushes to a shared repository. It covers the CI feedback loop, build automation, automated testing, artifact management, code quality gates, and CI server architecture for teams of any size.

### Target Audience

- **Developers** writing code that flows through CI pipelines daily
- **DevOps Engineers** designing and maintaining CI infrastructure and workflows
- **Platform Engineers** building internal developer platforms with CI at the core
- **Engineering Managers** establishing CI practices, metrics, and quality standards

### Scope

- History, definition, and core principles of Continuous Integration
- The commit → build → test → report feedback loop
- Build systems, reproducible builds, and caching strategies
- Source code integration patterns: merge frequency, branches, and PR validation
- Automated testing tiers: unit tests, integration tests, coverage, and reporting
- Artifact versioning, repositories (Nexus, Artifactory, GitHub Packages), and lifecycle
- Code quality gates: linting, static analysis, and coverage thresholds
- CI server architecture: agents, queues, parallelism, self-hosted vs cloud
- Monorepo CI strategies and dependency management

---

## What Is Continuous Integration

**Continuous Integration (CI)** is a software development practice where developers integrate their work into a shared repository frequently — at minimum once per day — and each integration is verified by an automated build and automated tests to detect errors as quickly as possible.

Martin Fowler's canonical definition:

> "Continuous Integration is a software development practice where members of a team integrate their work frequently, usually each person integrates at least daily — leading to multiple integrations per day. Each integration is verified by an automated build (including test) to detect integration errors as quickly as possible."

### History and Origins

CI evolved through decades of software engineering practice:

| Era | Practice | Key Contribution |
|---|---|---|
| **1991** | Grady Booch coins "continuous integration" | First use of the term in object-oriented design |
| **1997** | Kent Beck and Extreme Programming (XP) | CI becomes a core XP practice |
| **2000** | CruiseControl released | First dedicated CI server (open source, Java-based) |
| **2006** | Martin Fowler publishes CI article | Defines the modern CI practice and principles |
| **2011** | Jenkins 1.0 (fork of Hudson) | Dominant open-source CI server for a decade |
| **2014** | Travis CI popularizes hosted CI | SaaS CI becomes mainstream for open-source projects |
| **2018** | GitHub Actions beta | CI integrated directly into source code hosting |
| **2020+** | Cloud-native CI (Tekton, Dagger) | CI pipelines as code, containerized build steps |

### Core Principles

The foundational principles of CI, as established by the XP and DevOps communities:

1. **Maintain a single source repository** — All code lives in one version-controlled repository that the team shares
2. **Automate the build** — A single command should produce a working build from source
3. **Make the build self-testing** — The build process must run automated tests; a build that compiles but fails tests is a broken build
4. **Everyone commits to the mainline every day** — Frequent integration reduces merge conflicts and integration risk
5. **Every commit triggers a build** — Automation detects problems within minutes of introduction
6. **Fix broken builds immediately** — A broken build is the team's highest-priority issue
7. **Keep the build fast** — The feedback loop must complete in minutes, not hours
8. **Test in a clone of production** — CI environments should mirror production as closely as possible
9. **Make it easy to get the latest deliverables** — Anyone should be able to obtain the most recent build
10. **Everyone can see what is happening** — Build status, test results, and metrics are visible to the entire team

### CI vs Continuous Delivery vs Continuous Deployment

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   Continuous Integration (CI)                                                │
│   ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐        │
│   │   Commit    │──▶│   Build    │──▶│   Test     │──▶│  Artifact  │        │
│   └────────────┘   └────────────┘   └────────────┘   └────────────┘        │
│                                                              │               │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─ ─   │
│                                                              │               │
│   Continuous Delivery (CD)                                   ▼               │
│   ┌────────────┐   ┌────────────┐   ┌──────────────────────┐               │
│   │  Stage     │──▶│  Approval  │──▶│  Production Deploy   │               │
│   │  Deploy    │   │  (manual)  │   │  (manual trigger)    │               │
│   └────────────┘   └────────────┘   └──────────────────────┘               │
│                                                                              │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─   │
│                                                                              │
│   Continuous Deployment                                                      │
│   ┌────────────┐   ┌──────────────────────┐                                 │
│   │  Stage     │──▶│  Production Deploy   │  ← Every passing commit goes   │
│   │  Deploy    │   │  (automatic)         │    straight to production       │
│   └────────────┘   └──────────────────────┘                                 │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

| Aspect | Continuous Integration | Continuous Delivery | Continuous Deployment |
|---|---|---|---|
| **Scope** | Build + test on every commit | CI + deploy to staging automatically | CI + deploy to production automatically |
| **Manual step** | None (within CI) | Manual approval to production | None — fully automated |
| **Risk** | Low — catches bugs early | Medium — requires staging validation | Requires mature testing and monitoring |
| **Adoption prerequisite** | Version control, automated tests | CI in place, staging environment | Continuous Delivery in place, feature flags |

---

## The CI Feedback Loop

The CI feedback loop is the heartbeat of modern software development. Every code change triggers a deterministic sequence: **commit → build → test → report**. The speed and reliability of this loop directly impacts developer productivity.

```
                     Developer Workstation
                            │
                            ▼
                    ┌───────────────┐
                    │  git commit   │
                    │  git push     │
                    └───────┬───────┘
                            │
                            ▼
               ┌────────────────────────┐
               │     CI Server          │
               │  ┌──────────────────┐  │
               │  │  1. Clone repo   │  │
               │  │  2. Install deps │  │
               │  │  3. Build        │──┼──▶  Build logs
               │  │  4. Run tests    │──┼──▶  Test results
               │  │  5. Quality gate │──┼──▶  Coverage report
               │  │  6. Publish      │──┼──▶  Artifact
               │  └──────────────────┘  │
               └────────────┬───────────┘
                            │
                            ▼
               ┌────────────────────────┐
               │      Feedback          │
               │  ✅ Slack / Email      │
               │  ✅ PR status check    │
               │  ✅ Dashboard          │
               └────────────────────────┘
```

### Commit

The loop starts when a developer pushes a commit (or opens a pull request). The CI system detects the change via a **webhook** — a push notification from the version control system to the CI server.

```yaml
# GitHub Actions — trigger on push and pull request
name: CI
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
```

### Build

The build step compiles source code, resolves dependencies, and produces runnable output. A build should be:

- **Deterministic** — the same inputs always produce the same outputs
- **Isolated** — no state leaks between builds
- **Fast** — ideally under 5 minutes for the build step alone

```yaml
jobs:
  build:
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

      - name: Build
        run: npm run build
```

### Test

Automated tests validate that the build is correct. Tests in CI typically run in tiers:

1. **Unit tests** — fast, isolated, run first
2. **Integration tests** — test component interactions
3. **End-to-end tests** — validate full user flows (often in a later pipeline stage)

```yaml
      - name: Run unit tests
        run: npm test -- --coverage --ci

      - name: Run integration tests
        run: npm run test:integration
```

### Report

The final step is reporting results back to developers. Effective CI reporting includes:

- **Commit status checks** on the pull request (pass/fail badge)
- **Inline annotations** pointing to the exact lines that caused failures
- **Test result summaries** with counts of passed, failed, and skipped tests
- **Coverage reports** showing which lines are tested
- **Build logs** for debugging failures

```yaml
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: coverage/

      - name: Publish coverage
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage/lcov.info
```

### The Fast Feedback Imperative

Build speed is not a luxury — it directly affects how often developers integrate code.

| Build Time | Developer Behavior | Integration Frequency |
|---|---|---|
| **< 5 min** | Wait for result, fix immediately | Multiple times per day |
| **5–15 min** | Context-switch to another task | A few times per day |
| **15–30 min** | Batch multiple changes before pushing | Once per day |
| **> 30 min** | Avoid running CI, push large changes | Rarely — CI value collapses |

Target: **under 10 minutes** for the full CI pipeline from commit to green status check.

---

## Build Automation

Build automation is the foundation of CI. Without a reliable, repeatable build, nothing else in the pipeline can be trusted.

### Build Systems

Every language ecosystem has its own build tooling. The CI pipeline invokes these tools in a standardized way.

| Language | Build Tool | Build Command | Output |
|---|---|---|---|
| **Java** | Maven / Gradle | `mvn package` / `gradle build` | `.jar` / `.war` |
| **Go** | `go build` | `go build -o ./bin/app ./cmd/app` | Static binary |
| **Node.js** | npm / pnpm | `npm run build` | `dist/` directory |
| **Python** | setuptools / poetry | `poetry build` | `.whl` / `.tar.gz` |
| **C# / .NET** | dotnet CLI | `dotnet build` / `dotnet publish` | `.dll` / self-contained binary |
| **Rust** | Cargo | `cargo build --release` | Static binary |
| **C/C++** | Make / CMake | `cmake --build .` | Binary / shared library |

### Reproducible Builds

A reproducible build guarantees that the same source code, environment, and dependencies always produce a bit-for-bit identical output. This is critical for auditability, security, and debugging.

Techniques for reproducible builds:

1. **Pin dependency versions** — Use lock files (`package-lock.json`, `go.sum`, `Cargo.lock`)
2. **Pin tool versions** — Specify exact compiler and runtime versions in CI
3. **Use containerized builds** — Run builds inside Docker images with fixed toolchains
4. **Eliminate non-determinism** — Remove timestamps, randomized orderings, and host-specific paths from build output
5. **Vendor dependencies** — Store dependencies in the repository to avoid network-dependent builds

```yaml
# Pinning tool versions in GitHub Actions
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22.4'  # Pin exact version, not '1.22.x'

      - name: Build
        run: go build -trimpath -ldflags="-s -w" -o ./bin/app ./cmd/app
```

### Build Caching

Caching dramatically reduces build times by reusing work from previous runs. Effective CI caching targets:

- **Dependency caches** — `node_modules/`, `.m2/repository/`, `~/.cache/go-build/`
- **Build output caches** — compiled object files, intermediate artifacts
- **Docker layer caches** — reuse image layers across builds

```
Without caching:                     With caching:

┌─────────────────────┐              ┌─────────────────────┐
│ Install deps   3 min│              │ Restore cache   5 sec│
│ Compile        2 min│              │ Install deps   10 sec│ (mostly cached)
│ Total:         5 min│              │ Compile        30 sec│ (incremental)
└─────────────────────┘              │ Save cache     10 sec│
                                     │ Total:        ~1 min │
                                     └─────────────────────┘
```

```yaml
# GitHub Actions — caching Node.js dependencies
- name: Cache node_modules
  uses: actions/cache@v4
  with:
    path: node_modules
    key: node-modules-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      node-modules-

# GitHub Actions — caching Go build and module cache
- name: Cache Go modules
  uses: actions/cache@v4
  with:
    path: |
      ~/go/pkg/mod
      ~/.cache/go-build
    key: go-${{ hashFiles('go.sum') }}
    restore-keys: |
      go-
```

### Build Automation Example

A complete build automation pipeline for a Go microservice:

```yaml
name: Build
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  GO_VERSION: '1.22.4'
  BINARY_NAME: 'myservice'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}

      - name: Download dependencies
        run: go mod download

      - name: Verify dependencies
        run: go mod verify

      - name: Build binary
        run: |
          CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
            go build \
              -trimpath \
              -ldflags="-s -w -X main.version=${{ github.sha }}" \
              -o ./bin/${{ env.BINARY_NAME }} \
              ./cmd/${{ env.BINARY_NAME }}

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: ${{ env.BINARY_NAME }}-linux-amd64
          path: ./bin/${{ env.BINARY_NAME }}
          retention-days: 7
```

---

## Source Code Integration

CI only works if developers actually integrate their code frequently. The branching strategy and merge workflow directly determine how much value CI provides.

### Merge Frequency

The frequency at which developers merge to the main branch is the single most important CI metric. Higher merge frequency means smaller changes, fewer conflicts, and faster feedback.

```
Low merge frequency (risky):

  main:     A─────────────────────────────────────────M  (big-bang merge)
              \                                      /
  feature:     B──C──D──E──F──G──H──I──J──K──L──M──N   (2 weeks of changes)


High merge frequency (safe):

  main:     A──M₁──M₂──M₃──M₄──M₅──M₆──M₇──M₈
              \  /  \  /  \  /  \  /
  feature:     B    D    F    H
```

| Merge Frequency | Avg Change Size | Conflict Risk | Debug Difficulty |
|---|---|---|---|
| **Multiple times/day** | 10–50 lines | Very low | Easy — small blast radius |
| **Once per day** | 50–200 lines | Low | Moderate |
| **Once per week** | 200–1000 lines | Medium | Hard |
| **Once per sprint** | 1000+ lines | High | Very hard — many interacting changes |

### Integration Branches

Teams use different branching models depending on their maturity and release cadence:

**Trunk-Based Development** (recommended for CI):

```
main:    A──B──C──D──E──F──G──H──I──J
          \  /   \  /       \  /
           SF₁    SF₂        SF₃       (short-lived feature branches)
```

- Developers work on short-lived branches (< 2 days)
- Merge to `main` (trunk) at least daily
- Feature flags hide incomplete work
- Releases are cut from trunk

**Git Flow** (longer release cycles):

```
main:       A─────────────────────────R₁
             \                       /
develop:      B──C──D──E──F──G──H──I
               \     / \         /
feature/x:      M──N    \      /
                         \    /
feature/y:                P──Q
```

- `develop` branch for integration
- Feature branches merged to `develop`
- Release branches cut from `develop`, merged to `main`
- Higher overhead, suitable for packaged software with scheduled releases

### Pull Request Validation

Pull requests (PRs) are the primary integration point in most CI workflows. CI runs **before merge**, giving reviewers confidence that the change is safe.

```yaml
# Required status checks — CI must pass before merge
name: PR Validation
on:
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npm run typecheck

      - name: Unit tests
        run: npm test -- --ci --coverage

      - name: Build
        run: npm run build
```

### Branch Protection Rules

Branch protection ensures that CI checks pass before code reaches the main branch. This is the enforcement mechanism for CI discipline.

| Rule | Purpose |
|---|---|
| **Require status checks to pass** | CI pipeline must succeed before merge |
| **Require branches to be up to date** | PR must include latest `main` changes before merge |
| **Require pull request reviews** | At least one approval from a team member |
| **Require signed commits** | Verify commit author identity |
| **Restrict force pushes** | Prevent history rewriting on protected branches |
| **Require linear history** | Enforce rebase or squash merges for clean history |

---

## Automated Testing in CI

Automated testing is the mechanism that gives CI its power. Without tests, a CI pipeline only proves that code compiles — not that it works.

### Unit Tests

Unit tests validate individual functions or classes in isolation. They are the fastest tests and form the base of the **testing pyramid**.

```
              ┌───────┐
             /  E2E    \            Slow, expensive, few
            /  Tests    \
           ├─────────────┤
          / Integration   \         Medium speed, moderate count
         /    Tests        \
        ├───────────────────┤
       /     Unit Tests      \      Fast, cheap, many
      /                       \
     └─────────────────────────┘
```

Characteristics of good unit tests in CI:

- **Fast** — the full unit test suite should complete in under 2 minutes
- **Isolated** — no database, network, or filesystem dependencies
- **Deterministic** — no flaky results; same input always yields same output
- **Comprehensive** — cover edge cases, error paths, and boundary conditions

```yaml
# Running unit tests with coverage in CI
- name: Run unit tests
  run: |
    go test -v -race -count=1 \
      -coverprofile=coverage.out \
      -covermode=atomic \
      ./...

- name: Check coverage threshold
  run: |
    COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
    echo "Total coverage: ${COVERAGE}%"
    if (( $(echo "$COVERAGE < 80" | bc -l) )); then
      echo "Coverage ${COVERAGE}% is below 80% threshold"
      exit 1
    fi
```

### Integration Tests

Integration tests validate that components work together correctly. They typically involve real databases, message queues, or API calls.

```yaml
# Integration tests with service containers (GitHub Actions)
jobs:
  integration-tests:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Run integration tests
        env:
          DATABASE_URL: postgres://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
        run: npm run test:integration
```

### Test Coverage

Test coverage measures how much of the codebase is exercised by tests. CI pipelines typically enforce minimum coverage thresholds and track coverage trends over time.

| Coverage Metric | What It Measures | Typical Threshold |
|---|---|---|
| **Line coverage** | Percentage of lines executed | 80% |
| **Branch coverage** | Percentage of conditional branches taken | 70% |
| **Function coverage** | Percentage of functions called | 90% |
| **Statement coverage** | Percentage of statements executed | 80% |

```yaml
# Uploading coverage to Codecov
- name: Upload coverage
  uses: codecov/codecov-action@v4
  with:
    files: ./coverage/lcov.info
    fail_ci_if_error: true
    flags: unittests
    token: ${{ secrets.CODECOV_TOKEN }}
```

**Coverage as a quality gate** — enforce that coverage does not decrease:

```yaml
# .codecov.yml — require coverage to not drop
coverage:
  status:
    project:
      default:
        target: auto       # Must not drop below previous level
        threshold: 1%      # Allow 1% fluctuation
    patch:
      default:
        target: 80%        # New code must have 80% coverage
```

### Test Reporting

Good test reporting transforms CI from a pass/fail gate into a diagnostic tool. Modern CI platforms support structured test reports.

```yaml
# JUnit XML test reporting in GitHub Actions
- name: Run tests with JUnit reporter
  run: |
    npm test -- --ci --reporters=default --reporters=jest-junit
  env:
    JEST_JUNIT_OUTPUT_DIR: ./reports
    JEST_JUNIT_OUTPUT_NAME: junit.xml

- name: Publish test results
  uses: dorny/test-reporter@v1
  if: always()
  with:
    name: Jest Tests
    path: ./reports/junit.xml
    reporter: jest-junit
```

### Test Parallelism

Large test suites benefit from parallel execution. CI platforms provide multiple strategies:

```
Sequential execution:               Parallel execution (4 workers):

┌──────────────────────────┐        ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐
│  Test 1 → Test 2 → ...  │        │ W1   │ │ W2   │ │ W3   │ │ W4   │
│  → Test N                │        │ T1   │ │ T2   │ │ T3   │ │ T4   │
│                          │        │ T5   │ │ T6   │ │ T7   │ │ T8   │
│  Total: N × avg_time    │        │ T9   │ │ T10  │ │ T11  │ │ T12  │
└──────────────────────────┘        └──────┘ └──────┘ └──────┘ └──────┘

                                     Total: ~(N / 4) × avg_time
```

```yaml
# GitHub Actions — matrix strategy for parallel test shards
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4

      - name: Run test shard
        run: |
          npm test -- --ci --shard=${{ matrix.shard }}/4
```

---

## Artifact Management

An **artifact** is any output of the CI build process that needs to be stored, versioned, and distributed. Proper artifact management ensures that what you tested is exactly what you deploy.

### What Is a Build Artifact

```
Source Code                 CI Pipeline                  Artifacts
┌──────────┐    ┌─────────────────────────┐    ┌─────────────────────┐
│ .go      │    │                         │    │ Binary executable   │
│ .java    │───▶│  Build   Test   Package │───▶│ Docker image        │
│ .ts      │    │                         │    │ npm package         │
│ .py      │    └─────────────────────────┘    │ Helm chart          │
└──────────┘                                   │ JAR / WAR file      │
                                               │ Documentation site  │
                                               └─────────────────────┘
```

Common artifact types:

| Artifact Type | Examples | Storage |
|---|---|---|
| **Compiled binaries** | `.exe`, ELF binaries, `.dll` | Artifact repository, GitHub Releases |
| **Container images** | Docker images, OCI images | Container registry (Docker Hub, ECR, GCR) |
| **Language packages** | `.jar`, `.whl`, npm tarballs | Maven Central, PyPI, npm registry |
| **Archives** | `.tar.gz`, `.zip` | Object storage (S3, GCS) |
| **Helm charts** | `.tgz` chart packages | Helm repository, OCI registry |
| **Documentation** | HTML site, PDF | Static hosting (GitHub Pages, S3) |

### Versioning Artifacts

Every artifact must be uniquely identifiable. Never overwrite a published artifact — treat artifacts as **immutable**.

Versioning strategies:

```
Semantic Versioning (releases):     Git SHA (CI builds):
  v1.0.0                              a1b2c3d4
  v1.0.1                              e5f6a7b8
  v1.1.0                              c9d0e1f2
  v2.0.0

Timestamp (nightly builds):         Branch + Build Number:
  2025-01-15T08-30-00Z                main-build-1234
  2025-01-16T08-30-00Z                feature-auth-build-56
```

```yaml
# Generating version metadata in CI
- name: Set version
  id: version
  run: |
    if [[ "${{ github.ref }}" == refs/tags/v* ]]; then
      echo "version=${GITHUB_REF#refs/tags/v}" >> "$GITHUB_OUTPUT"
    else
      SHORT_SHA=$(git rev-parse --short HEAD)
      echo "version=0.0.0-dev+${SHORT_SHA}" >> "$GITHUB_OUTPUT"
    fi

- name: Build with version
  run: |
    go build -ldflags="-X main.version=${{ steps.version.outputs.version }}" \
      -o ./bin/myapp ./cmd/myapp
```

### Artifact Repositories

Artifact repositories provide centralized storage, access control, and lifecycle management for build artifacts.

**GitHub Packages** — integrated with GitHub repositories:

```yaml
# Publishing a Docker image to GitHub Container Registry
- name: Log in to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    push: true
    tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

**JFrog Artifactory** — universal artifact repository:

```yaml
# Publishing a Maven artifact to Artifactory
- name: Publish to Artifactory
  run: |
    mvn deploy \
      -DaltDeploymentRepository=releases::https://mycompany.jfrog.io/artifactory/libs-release \
      -Dartifactory.username=${{ secrets.ARTIFACTORY_USER }} \
      -Dartifactory.password=${{ secrets.ARTIFACTORY_TOKEN }}
```

**Sonatype Nexus** — on-premises artifact repository:

```yaml
# Publishing an npm package to Nexus
- name: Configure npm registry
  run: |
    echo "@mycompany:registry=https://nexus.mycompany.com/repository/npm-releases/" > .npmrc
    echo "//nexus.mycompany.com/repository/npm-releases/:_authToken=${{ secrets.NEXUS_TOKEN }}" >> .npmrc

- name: Publish
  run: npm publish
```

### Artifact Repository Comparison

| Feature | GitHub Packages | JFrog Artifactory | Sonatype Nexus |
|---|---|---|---|
| **Hosting** | Cloud (GitHub) | Cloud or self-hosted | Self-hosted or cloud |
| **Supported formats** | npm, Maven, NuGet, Docker, RubyGems | 30+ package types | Maven, npm, Docker, PyPI, and more |
| **Access control** | GitHub permissions | Fine-grained RBAC | RBAC with LDAP/SAML |
| **Proxying/caching** | No | Yes — virtual repositories | Yes — proxy repositories |
| **License scanning** | Via Dependabot | Xray integration | Lifecycle integration |
| **Cost** | Free for public repos | Free tier, paid for enterprise | OSS edition free, Pro paid |
| **Best for** | GitHub-native workflows | Enterprise multi-format | On-premises / air-gapped |

---

## Code Quality Gates

Quality gates are automated checks that enforce code standards before a change can be merged. They transform subjective code review opinions into objective, repeatable measurements.

### Linting

Linters enforce code style, catch common mistakes, and prevent anti-patterns.

```yaml
# Multi-language linting in CI
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # JavaScript / TypeScript
      - name: ESLint
        run: npx eslint --max-warnings=0 'src/**/*.{ts,tsx}'

      # Python
      - name: Ruff
        run: |
          pip install ruff
          ruff check .
          ruff format --check .

      # Go
      - name: golangci-lint
        uses: golangci/golangci-lint-action@v4
        with:
          version: v1.57

      # Terraform
      - name: tflint
        run: |
          tflint --init
          tflint --recursive
```

### Static Analysis

Static analysis tools go deeper than linters — they detect bugs, security vulnerabilities, and code smells by analyzing code without executing it.

| Tool | Language | What It Finds |
|---|---|---|
| **SonarQube** | Multi-language | Bugs, vulnerabilities, code smells, duplication |
| **CodeQL** | Multi-language | Security vulnerabilities via semantic analysis |
| **Semgrep** | Multi-language | Custom pattern matching, security rules |
| **SpotBugs** | Java | Common Java bug patterns |
| **mypy** | Python | Type errors via static type checking |
| **Clippy** | Rust | Idiomatic Rust issues and common mistakes |

```yaml
# SonarQube analysis in CI
- name: SonarQube Scan
  uses: SonarSource/sonarqube-scan-action@v2
  env:
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

# CodeQL analysis in GitHub Actions
- name: Initialize CodeQL
  uses: github/codeql-action/init@v3
  with:
    languages: javascript, python

- name: Perform CodeQL Analysis
  uses: github/codeql-action/analyze@v3
```

### Code Coverage Thresholds

Code coverage thresholds prevent merging code that reduces test coverage. Set thresholds at two levels:

1. **Project-level** — overall repository coverage must not drop
2. **Patch-level** — new/changed code must meet a minimum coverage

```yaml
# Enforcing coverage thresholds in a Go project
- name: Check coverage
  run: |
    go test -coverprofile=coverage.out ./...
    TOTAL=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d '%')

    echo "## Coverage Report" >> "$GITHUB_STEP_SUMMARY"
    echo "Total coverage: **${TOTAL}%**" >> "$GITHUB_STEP_SUMMARY"

    if (( $(echo "$TOTAL < 80" | bc -l) )); then
      echo "::error::Coverage ${TOTAL}% is below the 80% threshold"
      exit 1
    fi
```

### Quality Gate Pipeline Example

A complete quality gate pipeline that enforces multiple standards:

```yaml
name: Quality Gates
on:
  pull_request:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for diff-based checks

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      # Gate 1 — Code formatting
      - name: Check formatting
        run: npx prettier --check 'src/**/*.{ts,tsx,json,css}'

      # Gate 2 — Linting (zero warnings)
      - name: Lint
        run: npx eslint --max-warnings=0 'src/**/*.{ts,tsx}'

      # Gate 3 — Type checking
      - name: Type check
        run: npx tsc --noEmit

      # Gate 4 — Unit tests with coverage
      - name: Test
        run: npx jest --ci --coverage --coverageThreshold='{"global":{"branches":70,"functions":80,"lines":80,"statements":80}}'

      # Gate 5 — Build verification
      - name: Build
        run: npm run build

      # Gate 6 — Bundle size check
      - name: Check bundle size
        run: npx size-limit
```

---

## CI Server Architecture

Understanding how CI servers work under the hood helps you design pipelines that are fast, reliable, and cost-effective.

### Agents and Runners

A CI system consists of a **controller** (scheduler) and one or more **agents** (workers) that execute build jobs.

```
┌──────────────────────────────────────────────────────────────────┐
│                          CI Controller                           │
│                                                                  │
│   ┌─────────────┐   ┌─────────────┐   ┌──────────────────────┐  │
│   │ Job Queue   │   │ Scheduler   │   │ Results Aggregator   │  │
│   │             │──▶│             │──▶│                      │  │
│   │ Job A       │   │ Match jobs  │   │ Store logs           │  │
│   │ Job B       │   │ to agents   │   │ Update status        │  │
│   │ Job C       │   │ by labels   │   │ Send notifications   │  │
│   └─────────────┘   └──────┬──────┘   └──────────────────────┘  │
│                            │                                     │
└────────────────────────────┼─────────────────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
       ┌────────────┐ ┌────────────┐ ┌────────────┐
       │  Agent 1   │ │  Agent 2   │ │  Agent 3   │
       │  linux/x64 │ │  linux/arm │ │  macos/arm │
       │  label:gpu │ │  label:std │ │  label:ios │
       └────────────┘ └────────────┘ └────────────┘
```

| CI Platform | Agent Term | Configuration |
|---|---|---|
| **GitHub Actions** | Runner | `runs-on: ubuntu-latest` |
| **GitLab CI** | Runner | `tags: [docker, linux]` |
| **Jenkins** | Agent / Node | `agent { label 'linux' }` |
| **Azure DevOps** | Agent | `pool: vmImage: 'ubuntu-latest'` |
| **CircleCI** | Executor | `docker: - image: cimg/node:20.0` |

### Build Queues

When more jobs are submitted than agents can handle, jobs wait in a queue. Queue management directly affects developer wait times.

```
High contention (queue buildup):

  Time ──────────────────────────────────────▶

  Agent 1: ████████████████████████████████████
  Agent 2: ████████████████████████████████████
  Queue:   ░░░░░░░░░░░░░░  (jobs waiting)
                            ↑
                            Developer waits here

Strategies to reduce queue time:
  1. Add more agents (scale horizontally)
  2. Reduce build duration (optimize pipelines)
  3. Use autoscaling runners (scale to demand)
  4. Prioritize PR builds over branch builds
```

### Parallelism and Concurrency

CI platforms offer multiple levels of parallelism:

**Job-level parallelism** — independent jobs run simultaneously:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  build:
    runs-on: ubuntu-latest
    needs: [lint, test]  # Runs after lint and test complete
    steps:
      - run: npm run build
```

```
Timeline:
  ┌──────┐
  │ lint │──────────┐
  └──────┘          │
  ┌──────┐          ▼
  │ test │──────▶┌───────┐
  └──────┘       │ build │
                 └───────┘
```

**Matrix builds** — run the same job across multiple configurations:

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node-version: [18, 20, 22]
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm test
```

This produces **9 parallel jobs** (3 operating systems × 3 Node versions).

### Self-Hosted vs Cloud-Hosted

| Aspect | Cloud-Hosted Runners | Self-Hosted Runners |
|---|---|---|
| **Setup** | Zero — managed by CI provider | Must provision and maintain machines |
| **Cost model** | Per-minute billing | Fixed infrastructure cost |
| **Scaling** | Automatic | Manual or requires autoscaler setup |
| **Build environment** | Ephemeral, clean each run | Persistent — risk of state leakage |
| **Network access** | Public internet only | Access to private networks, VPNs |
| **Hardware** | Standard VMs (2–8 vCPU) | Custom hardware (GPUs, high memory) |
| **Security** | Provider manages isolation | You manage isolation and hardening |
| **Best for** | Open source, small teams | Enterprise, compliance, specialized hardware |

```yaml
# Self-hosted runner with labels
jobs:
  build:
    runs-on: [self-hosted, linux, x64, gpu]
    steps:
      - uses: actions/checkout@v4
      - name: Train ML model
        run: python train.py --epochs 100
```

### CI Platform Comparison

| Feature | GitHub Actions | GitLab CI | Jenkins | CircleCI | Azure DevOps |
|---|---|---|---|---|---|
| **Config format** | YAML | YAML | Groovy (Jenkinsfile) | YAML | YAML |
| **Hosted runners** | ✅ | ✅ | ❌ (self-hosted only) | ✅ | ✅ |
| **Self-hosted** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Container jobs** | ✅ | ✅ (native) | ✅ (via plugin) | ✅ (native) | ✅ |
| **Matrix builds** | ✅ | ✅ (parallel keyword) | ✅ (matrix axis) | ✅ | ✅ |
| **Caching** | actions/cache | Built-in | Plugin-based | Built-in | Pipeline Caching |
| **Secrets mgmt** | Encrypted secrets | CI/CD variables | Credentials plugin | Contexts | Variable groups |
| **Marketplace** | 20,000+ actions | Templates | 1,800+ plugins | Orbs | Extensions |
| **Free tier** | 2,000 min/month | 400 min/month | Free (self-hosted) | 6,000 min/month | 1,800 min/month |

---

## Monorepo CI Considerations

A **monorepo** stores multiple projects or services in a single repository. CI in a monorepo must be selective — building and testing only what changed — to avoid wasteful full rebuilds on every commit.

### Change Detection

The core challenge: determine which projects are affected by a commit.

```
monorepo/
├── services/
│   ├── api/           ← changed
│   ├── web/
│   └── worker/
├── libs/
│   ├── common/        ← changed (api depends on this)
│   └── auth/
├── infra/
│   └── terraform/
└── .github/
    └── workflows/
```

If `libs/common/` changes, CI must rebuild `services/api/` (which depends on it) but **not** `services/web/` or `services/worker/` (if they do not depend on `libs/common/`).

### Selective Builds

Use path filters to trigger only relevant CI jobs:

```yaml
# GitHub Actions — path-based triggers
name: API Service CI
on:
  push:
    paths:
      - 'services/api/**'
      - 'libs/common/**'
      - '.github/workflows/api-ci.yml'
  pull_request:
    paths:
      - 'services/api/**'
      - 'libs/common/**'
```

For more complex dependency graphs, use a change detection tool:

```yaml
# Using dorny/paths-filter for conditional jobs
jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      api: ${{ steps.filter.outputs.api }}
      web: ${{ steps.filter.outputs.web }}
      worker: ${{ steps.filter.outputs.worker }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            api:
              - 'services/api/**'
              - 'libs/common/**'
            web:
              - 'services/web/**'
              - 'libs/common/**'
              - 'libs/auth/**'
            worker:
              - 'services/worker/**'
              - 'libs/common/**'

  build-api:
    needs: detect-changes
    if: needs.detect-changes.outputs.api == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building API service"

  build-web:
    needs: detect-changes
    if: needs.detect-changes.outputs.web == 'true'
    runs-on: ubuntu-latest
    steps:
      - run: echo "Building Web service"
```

### Monorepo CI Example

A complete monorepo CI pipeline using Nx (JavaScript/TypeScript) or Turborepo:

```yaml
# Turborepo-based monorepo CI
name: Monorepo CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Required for change detection

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      # Only build/test affected packages
      - name: Build affected
        run: npx turbo run build --filter='...[origin/main]'

      - name: Test affected
        run: npx turbo run test --filter='...[origin/main]'

      - name: Lint affected
        run: npx turbo run lint --filter='...[origin/main]'
```

---

## Dependency Management in CI

Dependencies are one of the most common sources of CI failures and security vulnerabilities. A disciplined approach to dependency management keeps builds reproducible and secure.

### Lock Files and Determinism

Lock files pin every dependency (including transitive dependencies) to exact versions. **Always commit lock files** and use install commands that respect them.

| Package Manager | Lock File | Deterministic Install Command |
|---|---|---|
| **npm** | `package-lock.json` | `npm ci` (not `npm install`) |
| **pnpm** | `pnpm-lock.yaml` | `pnpm install --frozen-lockfile` |
| **Yarn** | `yarn.lock` | `yarn install --frozen-lockfile` |
| **pip** | `requirements.txt` / `pip.lock` | `pip install -r requirements.txt --no-deps` |
| **Go** | `go.sum` | `go mod download` + `go mod verify` |
| **Cargo** | `Cargo.lock` | `cargo build --locked` |
| **Maven** | (pom.xml with fixed versions) | `mvn dependency:resolve` |

```yaml
# Correct — uses lock file, fails if out of sync
- name: Install dependencies
  run: npm ci

# Incorrect — may update lock file, non-deterministic
# - name: Install dependencies
#   run: npm install
```

### Dependency Caching

Cache dependencies between CI runs to avoid downloading them every time.

```yaml
# GitHub Actions — built-in caching for common ecosystems
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'  # Automatically caches ~/.npm based on package-lock.json

# Manual caching for custom paths
- name: Cache Maven repository
  uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      maven-

# Cache with multiple paths
- name: Cache Gradle
  uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
    restore-keys: |
      gradle-
```

### Automated Dependency Updates

Automated tools keep dependencies up to date and flag security vulnerabilities:

| Tool | Ecosystem | Features |
|---|---|---|
| **Dependabot** | npm, pip, Maven, Go, Docker, and more | Automated PRs for version updates, security alerts |
| **Renovate** | 50+ package managers | Highly configurable, auto-merge, grouping, scheduling |
| **Snyk** | Multi-language | Vulnerability scanning, fix PRs, license compliance |

```yaml
# .github/dependabot.yml — Dependabot configuration
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    open-pull-requests-limit: 10
    reviewers:
      - "platform-team"
    labels:
      - "dependencies"
    groups:
      dev-dependencies:
        dependency-type: "development"
        update-types:
          - "minor"
          - "patch"

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

---

## Next Steps

Continue your CI/CD learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | CI/CD Overview | Core CI/CD concepts, pipeline anatomy, and tool landscape |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Continuous Integration documentation |
