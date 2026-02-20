# GitHub Actions

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [What Are GitHub Actions](#what-are-github-actions)
   - [Event-Driven Automation Platform](#event-driven-automation-platform)
   - [GitHub Actions vs Traditional CI/CD](#github-actions-vs-traditional-cicd)
3. [Core Concepts](#core-concepts)
   - [Workflows](#workflows)
   - [Events](#events)
   - [Jobs](#jobs)
   - [Steps](#steps)
   - [Actions](#actions)
   - [Runners](#runners)
4. [Workflow Syntax and YAML Structure](#workflow-syntax-and-yaml-structure)
   - [Minimal Workflow](#minimal-workflow)
   - [Complete Workflow Anatomy](#complete-workflow-anatomy)
   - [Expressions and Contexts](#expressions-and-contexts)
5. [Triggers](#triggers)
   - [push](#push)
   - [pull_request](#pull_request)
   - [schedule](#schedule)
   - [workflow_dispatch](#workflow_dispatch)
   - [repository_dispatch](#repository_dispatch)
   - [Trigger Comparison](#trigger-comparison)
6. [Actions Marketplace](#actions-marketplace)
   - [Using Community Actions](#using-community-actions)
   - [Creating Custom Actions](#creating-custom-actions)
   - [JavaScript vs Docker vs Composite Actions](#javascript-vs-docker-vs-composite-actions)
7. [Runners](#runners-1)
   - [GitHub-Hosted Runners](#github-hosted-runners)
   - [Self-Hosted Runners](#self-hosted-runners)
   - [Runner Groups and Labels](#runner-groups-and-labels)
   - [Runner Comparison](#runner-comparison)
8. [Secrets and Environment Variables](#secrets-and-environment-variables)
   - [Repository Secrets](#repository-secrets)
   - [Environment Secrets](#environment-secrets)
   - [Organization Secrets](#organization-secrets)
   - [Configuration Variables](#configuration-variables)
9. [Reusable Workflows](#reusable-workflows)
   - [workflow_call Trigger](#workflow_call-trigger)
   - [Inputs and Outputs](#inputs-and-outputs)
   - [Secrets Inheritance](#secrets-inheritance)
   - [Nesting Reusable Workflows](#nesting-reusable-workflows)
10. [Composite Actions](#composite-actions)
    - [Composite Action Structure](#composite-action-structure)
    - [Reusable Workflows vs Composite Actions](#reusable-workflows-vs-composite-actions)
11. [Matrix Strategies](#matrix-strategies)
    - [Basic Matrix](#basic-matrix)
    - [Include and Exclude](#include-and-exclude)
    - [Dynamic Matrix](#dynamic-matrix)
12. [Caching](#caching)
    - [actions/cache](#actionscache)
    - [Dependency Caching Strategies](#dependency-caching-strategies)
    - [Cache Limits and Eviction](#cache-limits-and-eviction)
13. [Artifacts](#artifacts)
    - [Upload Artifacts](#upload-artifacts)
    - [Download Artifacts](#download-artifacts)
    - [Retention Policies](#retention-policies)
14. [Environments and Deployment Protection Rules](#environments-and-deployment-protection-rules)
    - [Environment Configuration](#environment-configuration)
    - [Protection Rules](#protection-rules)
    - [Deployment Workflow](#deployment-workflow)
15. [Concurrency Control](#concurrency-control)
    - [Concurrency Groups](#concurrency-groups)
    - [Cancel-in-Progress](#cancel-in-progress)
16. [Security Considerations](#security-considerations)
    - [GITHUB_TOKEN Permissions](#github_token-permissions)
    - [Third-Party Action Pinning](#third-party-action-pinning)
    - [OpenID Connect (OIDC)](#openid-connect-oidc)
    - [Security Hardening Checklist](#security-hardening-checklist)
17. [Next Steps](#next-steps)
18. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **GitHub Actions** — GitHub's built-in automation platform for CI/CD, testing, deployments, and any event-driven workflow. It covers the core concepts, workflow syntax, triggers, runners, reusable patterns, security hardening, and practical examples for teams building modern software delivery pipelines directly within GitHub.

### Target Audience

- **Developers** automating builds, tests, and deployments from their GitHub repositories
- **DevOps Engineers** designing CI/CD pipelines with GitHub-native tooling
- **Platform Engineers** building reusable workflow libraries and self-hosted runner infrastructure
- **SREs** securing deployment pipelines and managing environment protections

### Scope

- GitHub Actions architecture and event-driven execution model
- Workflow YAML syntax, triggers, and expression contexts
- GitHub-hosted and self-hosted runner configuration
- Reusable workflows, composite actions, and matrix strategies
- Caching, artifacts, environments, and deployment protections
- Security hardening: token permissions, action pinning, and OIDC

---

## What Are GitHub Actions

### Event-Driven Automation Platform

GitHub Actions is an **event-driven automation platform** integrated directly into GitHub. Every activity in a repository — pushing code, opening a pull request, creating an issue, publishing a release — can trigger automated workflows that build, test, package, deploy, or perform any custom task.

```
GitHub Actions Architecture
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  GitHub Repository
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │   Events              Workflows             Runners      │
  │   ┌──────────┐        ┌──────────────┐      ┌────────┐  │
  │   │ push     │        │ .github/     │      │ GitHub │  │
  │   │ PR       │──────▶ │  workflows/  │────▶ │ hosted │  │
  │   │ schedule │        │   ci.yml     │      │ runner │  │
  │   │ dispatch │        │   deploy.yml │      ├────────┤  │
  │   │ release  │        └──────────────┘      │ Self-  │  │
  │   └──────────┘               │              │ hosted │  │
  │                              │              │ runner │  │
  │                              ▼              └────────┘  │
  │                       ┌──────────────┐                  │
  │                       │ Jobs → Steps │                  │
  │                       │  → Actions   │                  │
  │                       └──────────────┘                  │
  └──────────────────────────────────────────────────────────┘
```

Unlike external CI/CD systems that require webhook configuration, OAuth tokens, and separate dashboards, GitHub Actions is a **first-party citizen** of the GitHub platform. Workflows live alongside your code in `.github/workflows/`, have native access to repository context (commits, PRs, issues), and their results appear directly in pull request checks and the Actions tab.

### GitHub Actions vs Traditional CI/CD

| Aspect | GitHub Actions | Traditional CI/CD (Jenkins, etc.) |
|---|---|---|
| **Configuration** | YAML in `.github/workflows/` | Jenkinsfile, web UI, or DSL |
| **Infrastructure** | GitHub-hosted runners included | Self-managed servers required |
| **Integration** | Native GitHub events and API | Webhook + plugin configuration |
| **Marketplace** | 20,000+ community actions | Plugin ecosystem varies |
| **Scaling** | Auto-scales with GitHub-hosted runners | Manual capacity management |
| **Pricing** | Free tier for public repos | License + infrastructure costs |
| **Secrets** | Built-in encrypted secrets | Plugin-dependent (Vault, etc.) |

---

## Core Concepts

### Workflows

A **workflow** is an automated process defined in a YAML file under `.github/workflows/`. Each workflow:

- Is triggered by one or more **events**
- Contains one or more **jobs**
- Runs on a specified **runner**

```
.github/
└── workflows/
    ├── ci.yml          # Runs on every push and PR
    ├── deploy.yml      # Deploys on release
    ├── nightly.yml     # Scheduled nightly builds
    └── cleanup.yml     # Runs on workflow_dispatch
```

### Events

Events are specific activities that trigger a workflow. GitHub supports over 35 event types:

| Category | Events |
|---|---|
| **Code** | `push`, `pull_request`, `pull_request_target`, `create`, `delete` |
| **Issues** | `issues`, `issue_comment`, `label` |
| **Releases** | `release`, `registry_package` |
| **Scheduled** | `schedule` (cron syntax) |
| **Manual** | `workflow_dispatch`, `repository_dispatch` |
| **Workflow** | `workflow_run`, `workflow_call` |

### Jobs

A **job** is a set of steps that execute on the same runner. Jobs run in parallel by default, but you can define dependencies between them:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm test

  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying..."
```

```
Job Dependency Graph
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────┐
  │  build  │
  └────┬────┘
       │
  ┌────▼────┐
  │  test   │
  └────┬────┘
       │
  ┌────▼────┐
  │ deploy  │
  └─────────┘
```

### Steps

Steps are individual tasks within a job. Each step either runs a shell command (`run`) or uses an action (`uses`):

```yaml
steps:
  # Step using an action
  - name: Checkout repository
    uses: actions/checkout@v4

  # Step running a shell command
  - name: Install dependencies
    run: npm ci

  # Step with environment variables
  - name: Run tests
    run: npm test
    env:
      CI: true
      NODE_ENV: test
```

### Actions

Actions are **reusable units of code** that perform a specific task. They are the building blocks of workflows:

- **JavaScript actions** — Run directly on the runner using Node.js
- **Docker actions** — Run inside a Docker container
- **Composite actions** — Combine multiple steps into a single action

### Runners

Runners are the **compute environments** where jobs execute. GitHub provides hosted runners, or you can host your own:

- **GitHub-hosted** — Managed VMs (Ubuntu, Windows, macOS) spun up fresh for every job
- **Self-hosted** — Your own machines registered with GitHub, running the runner agent

---

## Workflow Syntax and YAML Structure

### Minimal Workflow

The simplest possible workflow requires a name, a trigger, and at least one job:

```yaml
name: Hello World

on: push

jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Hello, GitHub Actions!"
```

### Complete Workflow Anatomy

A full-featured workflow demonstrates the breadth of configuration options:

```yaml
name: CI Pipeline

# Triggers
on:
  push:
    branches: [main, develop]
    paths:
      - 'src/**'
      - 'package.json'
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

# Workflow-level permissions
permissions:
  contents: read
  pull-requests: write

# Workflow-level environment variables
env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io

# Workflow-level concurrency
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    name: Lint and Format
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

  test:
    name: Test (${{ matrix.os }}, Node ${{ matrix.node }})
    needs: lint
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: ['18', '20', '22']
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'

      - run: npm ci
      - run: npm test

      - name: Upload coverage
        if: matrix.os == 'ubuntu-latest' && matrix.node == '20'
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
          retention-days: 7

  build:
    name: Build
    needs: test
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - run: npm ci
      - run: npm run build

      - name: Get version
        id: version
        run: echo "version=$(node -p "require('./package.json').version")" >> "$GITHUB_OUTPUT"

      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/
```

### Expressions and Contexts

GitHub Actions provides expressions for dynamic values and conditional logic. Expressions are wrapped in `${{ }}`:

```yaml
steps:
  # Context variables
  - run: echo "Repository is ${{ github.repository }}"
  - run: echo "Branch is ${{ github.ref_name }}"
  - run: echo "Actor is ${{ github.actor }}"

  # Conditional execution
  - name: Deploy to production
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    run: ./deploy.sh

  # Status check functions
  - name: Always run cleanup
    if: always()
    run: ./cleanup.sh

  - name: Notify on failure
    if: failure()
    run: ./notify-slack.sh

  # Combining conditions
  - name: Annotate PR
    if: github.event_name == 'pull_request' && contains(github.event.pull_request.labels.*.name, 'deploy')
    run: echo "Deploying PR preview..."
```

| Context | Description | Example |
|---|---|---|
| `github` | Workflow run and event info | `github.sha`, `github.ref`, `github.actor` |
| `env` | Environment variables | `env.NODE_VERSION` |
| `vars` | Configuration variables | `vars.DEPLOY_URL` |
| `job` | Current job info | `job.status` |
| `steps` | Step outputs and status | `steps.build.outputs.version` |
| `runner` | Runner machine info | `runner.os`, `runner.arch` |
| `secrets` | Encrypted secrets | `secrets.DEPLOY_KEY` |
| `matrix` | Matrix parameters | `matrix.os`, `matrix.node` |
| `needs` | Outputs from dependent jobs | `needs.build.outputs.version` |
| `inputs` | Workflow/action inputs | `inputs.environment` |

---

## Triggers

### push

Triggers when commits are pushed to matching branches or tags:

```yaml
on:
  push:
    # Branch filters
    branches:
      - main
      - 'release/**'
    branches-ignore:
      - 'feature/experimental-*'

    # Tag filters
    tags:
      - 'v*.*.*'
    tags-ignore:
      - 'v*-beta'

    # Path filters — only trigger when these files change
    paths:
      - 'src/**'
      - 'package.json'
      - '!src/**/*.test.js'    # Negation pattern
```

### pull_request

Triggers on pull request activity. By default, runs for `opened`, `synchronize`, and `reopened` types:

```yaml
on:
  pull_request:
    branches: [main]
    types:
      - opened
      - synchronize
      - reopened
      - ready_for_review
    paths:
      - 'src/**'
      - 'tests/**'
```

> **Important:** `pull_request` workflows from forks run with read-only `GITHUB_TOKEN` and cannot access secrets. Use `pull_request_target` for trusted workflows that need write access, but exercise extreme caution — the workflow YAML comes from the base branch but can check out untrusted fork code.

### schedule

Triggers on a cron schedule (UTC timezone):

```yaml
on:
  schedule:
    # Every day at 2:00 AM UTC
    - cron: '0 2 * * *'

    # Every Monday at 9:00 AM UTC
    - cron: '0 9 * * 1'

    # Every 6 hours
    - cron: '0 */6 * * *'
```

```
Cron Syntax
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6, Sun=0)
│ │ │ │ │
* * * * *
```

> **Note:** Scheduled workflows run on the default branch only and may be delayed during periods of high load on GitHub Actions.

### workflow_dispatch

Enables manual triggering from the GitHub UI or API, with optional input parameters:

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target deployment environment'
        required: true
        type: choice
        options:
          - staging
          - production
      dry_run:
        description: 'Run in dry-run mode'
        required: false
        type: boolean
        default: false
      version:
        description: 'Version to deploy'
        required: false
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: |
          echo "Environment: ${{ inputs.environment }}"
          echo "Dry run: ${{ inputs.dry_run }}"
          echo "Version: ${{ inputs.version }}"
```

### repository_dispatch

Triggers via an external HTTP request to the GitHub API, enabling integration with third-party systems:

```yaml
on:
  repository_dispatch:
    types: [deploy, rollback]

jobs:
  handle-dispatch:
    runs-on: ubuntu-latest
    steps:
      - name: Handle event
        run: |
          echo "Event type: ${{ github.event.action }}"
          echo "Payload: ${{ toJson(github.event.client_payload) }}"
```

```bash
# Trigger via GitHub API
curl -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/OWNER/REPO/dispatches \
  -d '{"event_type":"deploy","client_payload":{"env":"production","ref":"v1.2.3"}}'
```

### Trigger Comparison

| Trigger | Use Case | Configurable Inputs | Branch Filter |
|---|---|---|---|
| `push` | Build on every commit | No | Yes |
| `pull_request` | Validate PR changes | No | Yes |
| `schedule` | Nightly builds, maintenance | No | Default branch only |
| `workflow_dispatch` | Manual deployments | Yes (UI/API) | Yes |
| `repository_dispatch` | External system integration | Yes (payload) | Default branch only |
| `workflow_run` | Chain workflows | No | Yes |
| `release` | Publish packages on release | No | No (tag-based) |

---

## Actions Marketplace

### Using Community Actions

The [GitHub Marketplace](https://github.com/marketplace?type=actions) hosts over 20,000 community-built actions. Reference them with `uses`:

```yaml
steps:
  # Official GitHub actions
  - uses: actions/checkout@v4
  - uses: actions/setup-node@v4
  - uses: actions/cache@v4

  # Community actions (owner/repo@ref)
  - uses: docker/build-push-action@v6
  - uses: aws-actions/configure-aws-credentials@v4

  # Pin to a specific commit SHA for security
  - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

  # Action from a subdirectory
  - uses: actions/aws/ec2@main

  # Action from a private repository (same org)
  - uses: my-org/private-actions/deploy@v1
```

### Creating Custom Actions

Custom actions are defined in a repository with an `action.yml` metadata file:

```yaml
# action.yml
name: 'Setup Project'
description: 'Install dependencies and build the project'
inputs:
  node-version:
    description: 'Node.js version to use'
    required: false
    default: '20'
  working-directory:
    description: 'Working directory for the project'
    required: false
    default: '.'
outputs:
  build-version:
    description: 'The version that was built'
    value: ${{ steps.version.outputs.result }}
runs:
  using: 'composite'
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Install and build
      working-directory: ${{ inputs.working-directory }}
      shell: bash
      run: |
        npm ci
        npm run build

    - name: Get version
      id: version
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: echo "result=$(node -p "require('./package.json').version")" >> "$GITHUB_OUTPUT"
```

### JavaScript vs Docker vs Composite Actions

| Aspect | JavaScript | Docker | Composite |
|---|---|---|---|
| **Runtime** | Node.js on runner | Container on runner | Runner shell(s) |
| **Startup** | Fast (no container pull) | Slower (image pull) | Fast |
| **OS Support** | Linux, Windows, macOS | Linux only | Linux, Windows, macOS |
| **Dependencies** | Bundled with action | Inside container image | Installed by steps |
| **Best For** | GitHub API interactions | Complex toolchains | Reusing step sequences |
| **Distribution** | npm packages or bundled JS | Docker Hub or GHCR | action.yml in repo |

---

## Runners

### GitHub-Hosted Runners

GitHub provides managed virtual machines that are provisioned fresh for every job:

| Runner Label | OS | vCPUs | RAM | Storage |
|---|---|---|---|---|
| `ubuntu-latest` | Ubuntu 22.04 | 4 | 16 GB | 14 GB SSD |
| `ubuntu-24.04` | Ubuntu 24.04 | 4 | 16 GB | 14 GB SSD |
| `windows-latest` | Windows Server 2022 | 4 | 16 GB | 14 GB SSD |
| `macos-latest` | macOS 14 (Sonoma) | 3 (M1) | 7 GB | 14 GB SSD |
| `macos-13` | macOS 13 (Ventura) | 4 (Intel) | 14 GB | 14 GB SSD |

```yaml
jobs:
  linux-build:
    runs-on: ubuntu-latest

  windows-build:
    runs-on: windows-latest

  macos-build:
    runs-on: macos-latest

  # Larger runners (GitHub Teams/Enterprise)
  large-build:
    runs-on: ubuntu-latest-16-cores
```

GitHub-hosted runners come pre-installed with common tools: Docker, Node.js, Python, Java, .NET, Go, Rust, and more. Each job gets a clean VM, so there is no state pollution between jobs.

### Self-Hosted Runners

Self-hosted runners give you full control over hardware, OS, and installed software:

```bash
# Download and configure a self-hosted runner
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.321.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.321.0/actions-runner-linux-x64-2.321.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.321.0.tar.gz

# Configure
./config.sh --url https://github.com/OWNER/REPO --token YOUR_TOKEN

# Run as a service
sudo ./svc.sh install
sudo ./svc.sh start
```

```yaml
jobs:
  build:
    # Target self-hosted runners by labels
    runs-on: [self-hosted, linux, x64, gpu]
    steps:
      - uses: actions/checkout@v4
      - run: nvidia-smi  # GPU available on this runner
```

### Runner Groups and Labels

Runner groups control which repositories and workflows can use specific self-hosted runners:

```
Runner Organization
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Organization
  ┌────────────────────────────────────────────────────┐
  │                                                    │
  │  Runner Group: "production"                        │
  │  ┌──────────────────────────────────────────────┐  │
  │  │  runner-prod-01  [self-hosted, linux, prod]  │  │
  │  │  runner-prod-02  [self-hosted, linux, prod]  │  │
  │  └──────────────────────────────────────────────┘  │
  │                                                    │
  │  Runner Group: "gpu-builds"                        │
  │  ┌──────────────────────────────────────────────┐  │
  │  │  runner-gpu-01   [self-hosted, linux, gpu]   │  │
  │  └──────────────────────────────────────────────┘  │
  │                                                    │
  │  Runner Group: "default"                           │
  │  ┌──────────────────────────────────────────────┐  │
  │  │  runner-dev-01   [self-hosted, linux, dev]   │  │
  │  │  runner-dev-02   [self-hosted, linux, dev]   │  │
  │  └──────────────────────────────────────────────┘  │
  └────────────────────────────────────────────────────┘
```

### Runner Comparison

| Feature | GitHub-Hosted | Self-Hosted |
|---|---|---|
| **Maintenance** | Managed by GitHub | You manage updates and patches |
| **Clean environment** | Fresh VM every job | Persistent — must manage state |
| **Cost** | Included minutes + overage | Your infrastructure costs |
| **Customization** | Limited to pre-installed tools | Full control over software/hardware |
| **Network** | Public internet access | Access to private networks |
| **GPU/Specialized HW** | Not available (standard) | Full hardware access |
| **Startup time** | ~20-40 seconds | Near-instant (already running) |
| **Security** | Isolated VMs, no cross-job state | Shared machine risk if misconfigured |

---

## Secrets and Environment Variables

### Repository Secrets

Secrets are encrypted values accessible in workflows. They are masked in logs automatically:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to server
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
          API_TOKEN: ${{ secrets.API_TOKEN }}
        run: |
          echo "$DEPLOY_KEY" > deploy_key
          chmod 600 deploy_key
          ssh -i deploy_key user@server "deploy --token $API_TOKEN"
```

### Environment Secrets

Environment-scoped secrets are only available to jobs targeting that environment:

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - run: deploy --url ${{ secrets.DEPLOY_URL }}
        # DEPLOY_URL comes from the "staging" environment secrets

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: deploy --url ${{ secrets.DEPLOY_URL }}
        # DEPLOY_URL comes from the "production" environment secrets
```

### Organization Secrets

Organization-level secrets can be shared across repositories with visibility policies:

| Visibility | Access |
|---|---|
| All repositories | Every repo in the org |
| Private repositories | Only private repos |
| Selected repositories | Explicitly chosen repos |

### Configuration Variables

For non-sensitive values, use **configuration variables** (`vars` context) instead of secrets:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Use configuration variables
        run: |
          echo "App name: ${{ vars.APP_NAME }}"
          echo "Deploy URL: ${{ vars.DEPLOY_URL }}"
          echo "Region: ${{ vars.AWS_REGION }}"
```

| Feature | Secrets | Variables |
|---|---|---|
| **Encryption** | Encrypted at rest | Stored as plaintext |
| **Log masking** | Automatically masked | Not masked |
| **Use case** | Passwords, tokens, keys | URLs, feature flags, config |
| **Context** | `secrets.*` | `vars.*` |
| **Size limit** | 48 KB per secret | 48 KB per variable |

---

## Reusable Workflows

### workflow_call Trigger

Reusable workflows allow you to define a workflow once and call it from other workflows, reducing duplication across repositories:

```yaml
# .github/workflows/reusable-build.yml (the reusable workflow)
name: Reusable Build

on:
  workflow_call:
    inputs:
      node-version:
        description: 'Node.js version'
        required: false
        type: string
        default: '20'
      run-tests:
        description: 'Whether to run tests'
        required: false
        type: boolean
        default: true
    outputs:
      artifact-name:
        description: 'Name of the uploaded artifact'
        value: ${{ jobs.build.outputs.artifact-name }}
    secrets:
      NPM_TOKEN:
        required: false

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.upload.outputs.artifact-name }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: 'npm'

      - run: npm ci
        env:
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}

      - run: npm test
        if: inputs.run-tests

      - run: npm run build

      - name: Upload artifact
        id: upload
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ github.sha }}
          path: dist/
```

```yaml
# .github/workflows/ci.yml (the caller workflow)
name: CI

on:
  push:
    branches: [main]

jobs:
  call-build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      node-version: '20'
      run-tests: true
    secrets:
      NPM_TOKEN: ${{ secrets.NPM_TOKEN }}

  # Use output from the reusable workflow
  deploy:
    needs: call-build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Artifact: ${{ needs.call-build.outputs.artifact-name }}"
```

### Inputs and Outputs

Reusable workflows support typed inputs and outputs that create a clean interface:

| Input Type | Description | Example |
|---|---|---|
| `string` | Free-text string value | `'20'`, `'production'` |
| `number` | Numeric value | `3`, `100` |
| `boolean` | True or false | `true`, `false` |

Outputs from reusable workflows must be explicitly mapped from job outputs:

```yaml
on:
  workflow_call:
    outputs:
      image-tag:
        description: 'The Docker image tag that was built'
        value: ${{ jobs.docker.outputs.tag }}

jobs:
  docker:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.meta.outputs.tags }}
    steps:
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/my-org/my-app
```

### Secrets Inheritance

Instead of passing secrets one by one, you can inherit all secrets from the caller:

```yaml
jobs:
  call-deploy:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
    secrets: inherit    # Pass ALL caller secrets to the reusable workflow
```

> **Caution:** `secrets: inherit` passes every secret the caller has access to. For tighter control, pass secrets explicitly.

### Nesting Reusable Workflows

Reusable workflows can call other reusable workflows, up to 4 levels deep:

```
Caller Workflow
  └── Reusable Workflow (level 1)
        └── Reusable Workflow (level 2)
              └── Reusable Workflow (level 3)
                    └── Reusable Workflow (level 4) ← maximum depth
```

---

## Composite Actions

### Composite Action Structure

Composite actions bundle multiple steps into a single reusable action. Unlike reusable workflows, composite actions run **within** a job, not as a separate job:

```yaml
# .github/actions/setup-and-build/action.yml
name: 'Setup and Build'
description: 'Checkout, install, and build the project'
inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'
outputs:
  build-size:
    description: 'Size of the build output'
    value: ${{ steps.size.outputs.bytes }}
runs:
  using: 'composite'
  steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'

    - name: Install
      shell: bash
      run: npm ci

    - name: Build
      shell: bash
      run: npm run build

    - name: Measure build size
      id: size
      shell: bash
      run: echo "bytes=$(du -sb dist/ | cut -f1)" >> "$GITHUB_OUTPUT"
```

```yaml
# Usage in a workflow
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Setup and build
        uses: ./.github/actions/setup-and-build
        with:
          node-version: '20'

      - name: Report size
        run: echo "Build size: ${{ steps.setup-and-build.outputs.build-size }} bytes"
```

### Reusable Workflows vs Composite Actions

| Feature | Reusable Workflows | Composite Actions |
|---|---|---|
| **Runs as** | Separate job(s) | Steps within a calling job |
| **Runner** | Own `runs-on` per job | Inherits caller's runner |
| **Secrets access** | Via `secrets:` or `inherit` | Inherits caller's context |
| **Outputs** | Job-level outputs | Step-level outputs |
| **Nesting** | Up to 4 levels | Can use other actions |
| **Trigger** | `workflow_call` | `uses:` in a step |
| **Best for** | Complete pipelines | Shared step sequences |
| **Visibility** | Appears as separate jobs in UI | Steps appear inline |

---

## Matrix Strategies

### Basic Matrix

Matrix strategies generate multiple job runs with different parameter combinations:

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: ['18', '20', '22']
        # This creates 3 × 3 = 9 job runs
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci && npm test
```

```
Matrix Expansion (3 × 3 = 9 jobs)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────────┬──────────┬──────────┬──────────┐
  │                  │ Node 18  │ Node 20  │ Node 22  │
  ├──────────────────┼──────────┼──────────┼──────────┤
  │ ubuntu-latest    │   Job 1  │  Job 2   │  Job 3   │
  │ windows-latest   │   Job 4  │  Job 5   │  Job 6   │
  │ macos-latest     │   Job 7  │  Job 8   │  Job 9   │
  └──────────────────┴──────────┴──────────┴──────────┘
```

### Include and Exclude

Fine-tune the matrix with `include` (add combinations) and `exclude` (remove combinations):

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: ['18', '20']
    exclude:
      # Skip Node 18 on Windows
      - os: windows-latest
        node: '18'
    include:
      # Add a specific combination with extra variables
      - os: ubuntu-latest
        node: '22'
        experimental: true
      # Add an entry with entirely new values
      - os: macos-latest
        node: '20'
        test-runner: jest

  # Continue running other matrix jobs if one fails
  fail-fast: false

  # Limit concurrent matrix jobs
  max-parallel: 4
```

### Dynamic Matrix

Generate matrix values dynamically from a previous job:

```yaml
jobs:
  generate-matrix:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - uses: actions/checkout@v4
      - id: set-matrix
        run: |
          # Generate matrix from changed directories
          DIRS=$(find packages -maxdepth 1 -mindepth 1 -type d -printf '"%f",' | sed 's/,$//')
          echo "matrix={\"package\":[$DIRS]}" >> "$GITHUB_OUTPUT"

  test:
    needs: generate-matrix
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJson(needs.generate-matrix.outputs.matrix) }}
    steps:
      - uses: actions/checkout@v4
      - run: cd packages/${{ matrix.package }} && npm test
```

---

## Caching

### actions/cache

Caching dependencies between workflow runs dramatically speeds up builds:

```yaml
steps:
  - uses: actions/checkout@v4

  - name: Cache node modules
    id: cache-npm
    uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-npm-

  - name: Install dependencies
    if: steps.cache-npm.outputs.cache-hit != 'true'
    run: npm ci
```

### Dependency Caching Strategies

| Language | Cache Path | Key Strategy |
|---|---|---|
| **Node.js (npm)** | `~/.npm` | `hashFiles('**/package-lock.json')` |
| **Node.js (yarn)** | `~/.cache/yarn` | `hashFiles('**/yarn.lock')` |
| **Node.js (pnpm)** | `~/.local/share/pnpm/store` | `hashFiles('**/pnpm-lock.yaml')` |
| **Python (pip)** | `~/.cache/pip` | `hashFiles('**/requirements.txt')` |
| **Ruby (bundler)** | `vendor/bundle` | `hashFiles('**/Gemfile.lock')` |
| **Go** | `~/go/pkg/mod` | `hashFiles('**/go.sum')` |
| **Rust (cargo)** | `~/.cargo/registry`, `target/` | `hashFiles('**/Cargo.lock')` |
| **Java (Gradle)** | `~/.gradle/caches` | `hashFiles('**/*.gradle*', '**/gradle-wrapper.properties')` |
| **.NET (NuGet)** | `~/.nuget/packages` | `hashFiles('**/*.csproj')` |

Many `actions/setup-*` actions include built-in caching:

```yaml
# Built-in cache with setup-node
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'     # Automatically caches ~/.npm

# Built-in cache with setup-python
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'     # Automatically caches pip packages

# Built-in cache with setup-go
- uses: actions/setup-go@v5
  with:
    go-version: '1.22'
    cache: true      # Automatically caches Go modules

# Built-in cache with setup-java
- uses: actions/setup-java@v4
  with:
    distribution: 'temurin'
    java-version: '21'
    cache: 'gradle'  # Automatically caches Gradle dependencies
```

### Cache Limits and Eviction

| Limit | Value |
|---|---|
| **Maximum cache size** | 10 GB per repository |
| **Individual entry limit** | No per-entry limit (counts toward total) |
| **Eviction policy** | Least recently used (LRU) |
| **Retention** | Entries not accessed in 7 days are evicted |
| **Branch scope** | Caches from the default branch are available to all branches |

```
Cache Resolution Order
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  1. Exact key match on current branch
  2. restore-keys prefix match on current branch
  3. Exact key match on default branch
  4. restore-keys prefix match on default branch
```

---

## Artifacts

### Upload Artifacts

Artifacts persist data between jobs or after a workflow completes:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: webapp-build
          path: |
            dist/
            !dist/**/*.map
          retention-days: 5
          if-no-files-found: error    # fail if no files match
```

### Download Artifacts

Download artifacts in a subsequent job:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: webapp-build
          path: dist/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: webapp-build
          path: dist/

      - name: Deploy
        run: |
          ls -la dist/
          ./deploy.sh dist/

  download-all:
    needs: build
    runs-on: ubuntu-latest
    steps:
      # Download all artifacts from the workflow run
      - uses: actions/download-artifact@v4
        # Omit 'name' to download all artifacts
```

### Retention Policies

| Setting | Default | Range |
|---|---|---|
| **Default retention** | 90 days | — |
| **Custom retention** | `retention-days` input | 1–90 days |
| **Organization policy** | Can enforce max retention | 1–90 days |
| **Storage limit** | Counts toward GitHub storage quota | Varies by plan |

---

## Environments and Deployment Protection Rules

### Environment Configuration

Environments define deployment targets with specific secrets, variables, and protection rules:

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh staging

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh production
```

### Protection Rules

| Rule | Description |
|---|---|
| **Required reviewers** | One or more people must approve the deployment |
| **Wait timer** | Delay deployment by N minutes (up to 43,200 / 30 days) |
| **Branch restrictions** | Only specific branches can deploy to this environment |
| **Custom rules** | GitHub Apps can define custom protection rules |

```
Deployment Flow with Protection Rules
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Push to main
       │
       ▼
  ┌──────────┐     ┌──────────┐     ┌──────────────────┐
  │  Build   │────▶│  Test    │────▶│ Deploy Staging   │
  └──────────┘     └──────────┘     └────────┬─────────┘
                                             │
                                    ┌────────▼─────────┐
                                    │  Approval Gate   │
                                    │  (manual review) │
                                    └────────┬─────────┘
                                             │
                                    ┌────────▼─────────┐
                                    │   Wait Timer     │
                                    │   (15 minutes)   │
                                    └────────┬─────────┘
                                             │
                                    ┌────────▼─────────────┐
                                    │ Deploy Production    │
                                    └──────────────────────┘
```

### Deployment Workflow

A complete deployment workflow with environment protections:

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build
          path: dist/

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build
          path: dist/
      - name: Deploy to staging
        run: ./deploy.sh staging
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}

  smoke-test:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run smoke tests
        run: npm run test:smoke
        env:
          BASE_URL: https://staging.example.com

  deploy-production:
    needs: smoke-test
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build
          path: dist/
      - name: Deploy to production
        run: ./deploy.sh production
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

---

## Concurrency Control

### Concurrency Groups

Concurrency groups ensure only one workflow (or job) in the group runs at a time:

```yaml
# Workflow-level concurrency
concurrency:
  group: deploy-${{ github.ref }}

# Job-level concurrency
jobs:
  deploy:
    concurrency:
      group: deploy-production
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### Cancel-in-Progress

Cancel redundant runs when a new one starts (useful for CI on PRs):

```yaml
# Cancel outdated CI runs on the same PR
concurrency:
  group: ci-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
```

| Pattern | Group Key | Behavior |
|---|---|---|
| **One deploy per branch** | `deploy-${{ github.ref }}` | Queues per branch |
| **One deploy to production** | `deploy-production` | Global queue for prod |
| **Cancel stale PR checks** | `ci-${{ github.event.pull_request.number }}` | Cancels older PR runs |
| **One workflow per ref** | `${{ github.workflow }}-${{ github.ref }}` | Per-workflow per-branch |

---

## Security Considerations

### GITHUB_TOKEN Permissions

The `GITHUB_TOKEN` is automatically created for each workflow run. Apply the **principle of least privilege**:

```yaml
# Workflow-level: restrict default permissions
permissions:
  contents: read

jobs:
  lint:
    runs-on: ubuntu-latest
    # Job inherits workflow permissions
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint

  publish:
    runs-on: ubuntu-latest
    # Job-level: grant only what's needed
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

| Permission | Read | Write | Use Case |
|---|---|---|---|
| `contents` | Clone repo | Push commits, create releases | Most workflows |
| `pull-requests` | Read PRs | Comment, review, merge | PR automation |
| `issues` | Read issues | Create, comment, label | Issue triage bots |
| `packages` | Pull packages | Publish packages | Package publishing |
| `deployments` | Read deployments | Create deployments | CD pipelines |
| `id-token` | — | Request OIDC token | Cloud provider auth |
| `security-events` | Read alerts | Upload SARIF results | Code scanning |
| `actions` | Read workflows | Cancel, re-run workflows | Workflow management |

### Third-Party Action Pinning

Never reference actions by mutable tags alone. Pin to a full commit SHA for supply chain security:

```yaml
steps:
  # INSECURE — tag can be moved to malicious code
  - uses: actions/checkout@v4

  # SECURE — pinned to immutable commit SHA
  - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

  # Use Dependabot or Renovate to keep SHA pins up to date
  # .github/dependabot.yml
  # updates:
  #   - package-ecosystem: "github-actions"
  #     directory: "/"
  #     schedule:
  #       interval: "weekly"
```

### OpenID Connect (OIDC)

OIDC eliminates the need for long-lived cloud credentials by exchanging a short-lived GitHub token for a cloud provider session:

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # AWS — no stored access keys needed
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1

      # Azure — no stored service principal secrets
      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      # GCP — no stored service account keys
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/123456/locations/global/workloadIdentityPools/github/providers/my-repo'
          service_account: 'deploy@my-project.iam.gserviceaccount.com'
```

```
OIDC Authentication Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  GitHub Actions                        Cloud Provider
  ┌──────────────────┐                 ┌──────────────────┐
  │ 1. Request OIDC  │                 │                  │
  │    token from    │                 │                  │
  │    GitHub        │                 │                  │
  │                  │  2. JWT token   │                  │
  │    ─────────────────────────────▶  │ 3. Validate      │
  │                  │                 │    token claims  │
  │                  │  4. Short-lived │    (repo, branch │
  │    ◀─────────────────────────────  │     workflow)    │
  │                  │    credentials  │                  │
  │ 5. Use creds to │                 │                  │
  │    access cloud  │                 │                  │
  │    resources     │                 │                  │
  └──────────────────┘                 └──────────────────┘
```

### Security Hardening Checklist

1. **Set `permissions` at the workflow level** — Default to `contents: read` and grant only what each job needs.

2. **Pin actions to commit SHAs** — Use full SHA references instead of mutable tags. Automate updates with Dependabot.

3. **Use OIDC for cloud authentication** — Eliminate long-lived cloud credentials stored as secrets.

4. **Audit third-party actions** — Review source code before using community actions. Prefer actions from verified creators.

5. **Limit secret exposure** — Pass secrets as environment variables to specific steps, not entire jobs.

6. **Use environment protection rules** — Require approvals for production deployments. Restrict which branches can deploy.

7. **Be cautious with `pull_request_target`** — Never checkout and run untrusted PR code with write permissions.

8. **Enable Dependabot for GitHub Actions** — Keep action versions current and patched.

9. **Use `--no-cache` for sensitive builds** — Avoid cache poisoning attacks on build layers.

10. **Review workflow run logs** — Monitor for unexpected behavior, especially in workflows triggered by forks.

---

## Next Steps

Continue your CI/CD learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | CI/CD Overview | Core CI/CD concepts, pipeline anatomy, and tool landscape |
| [01-CONTINUOUS-INTEGRATION.md](01-CONTINUOUS-INTEGRATION.md) | Continuous Integration | Build automation, testing, artifact management, and quality gates |
| [02-CONTINUOUS-DELIVERY.md](02-CONTINUOUS-DELIVERY.md) | Continuous Delivery | Deployment pipelines, release strategies, and environment management |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial GitHub Actions documentation |
