# CI/CD Learning Resources

A comprehensive guide to Continuous Integration and Continuous Delivery/Deployment — from foundational concepts and pipeline design to GitOps, security scanning, and production-ready CI/CD patterns.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | CI/CD fundamentals, pipelines, DevOps culture | **Start here** |
| [01-CONTINUOUS-INTEGRATION](01-CONTINUOUS-INTEGRATION.md) | Build automation, testing in CI, artifact management | When setting up CI workflows |
| [02-CONTINUOUS-DELIVERY](02-CONTINUOUS-DELIVERY.md) | Deployment pipelines, environments, release strategies | When designing deployment flows |
| [03-GITHUB-ACTIONS](03-GITHUB-ACTIONS.md) | Workflows, actions, runners, reusable workflows | When using GitHub Actions |
| [04-AZURE-DEVOPS](04-AZURE-DEVOPS.md) | Azure Pipelines, YAML pipelines, service connections | When using Azure DevOps |
| [05-JENKINS](05-JENKINS.md) | Jenkinsfile, shared libraries, pipeline as code | When using Jenkins |
| [06-GITOPS](06-GITOPS.md) | ArgoCD, Flux, pull-based deployments, declarative config | When adopting GitOps practices |
| [07-TESTING-IN-PIPELINES](07-TESTING-IN-PIPELINES.md) | Unit/integration/E2E tests, test parallelization, quality gates | When integrating tests into pipelines |
| [08-SECURITY-IN-PIPELINES](08-SECURITY-IN-PIPELINES.md) | SAST, DAST, dependency scanning, supply chain security | **Essential — shift-left security** |
| [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Pipeline patterns, trunk-based development, feature flags | **Essential — production checklist** |
| [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | Common CI/CD mistakes and how to avoid them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured learning guide with exercises | **Start here** after the Overview |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand what CI/CD is and why it matters
   - Learn pipeline anatomy (stages, jobs, steps)
   - Explore DevOps culture and the CI/CD feedback loop

2. **Set Up Continuous Integration** ([01-CONTINUOUS-INTEGRATION](01-CONTINUOUS-INTEGRATION.md))
   - Automate builds triggered by code changes
   - Run tests on every commit or pull request
   - Publish and manage build artifacts

3. **Design Deployment Pipelines** ([02-CONTINUOUS-DELIVERY](02-CONTINUOUS-DELIVERY.md))
   - Define environments (dev, staging, production)
   - Implement release strategies (blue-green, canary, rolling)
   - Add approval gates and rollback mechanisms

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured curriculum with hands-on exercises
   - Progressive skill building from basics to production

### For Experienced Users

1. **Review Best Practices** ([09-BEST-PRACTICES](09-BEST-PRACTICES.md))
   - Production-ready pipeline patterns
   - Trunk-based development and feature flags
   - Monorepo and multi-service pipeline strategies

2. **Avoid Anti-Patterns** ([10-ANTI-PATTERNS](10-ANTI-PATTERNS.md))
   - Common pipeline mistakes and misconfigurations
   - Slow builds, flaky tests, and deployment pitfalls

3. **Secure Your Pipelines** ([08-SECURITY-IN-PIPELINES](08-SECURITY-IN-PIPELINES.md))
   - SAST, DAST, and dependency scanning
   - Supply chain security and signed artifacts
   - Secrets management and least-privilege runners

4. **Adopt GitOps** ([06-GITOPS](06-GITOPS.md))
   - Pull-based deployment with ArgoCD and Flux
   - Declarative configuration and drift detection
   - Git as the single source of truth for deployments

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Developer Workflow                         │
│   git commit │ git push │ pull request │ code review              │
└───────────────────────────┬─────────────────────────────────────┘
                            │ Webhook / Trigger
┌───────────────────────────▼─────────────────────────────────────┐
│                   Continuous Integration (CI)                     │
│   ┌─────────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│   │   Source Stage   │  │  Build Stage │  │   Test Stage      │  │
│   │  (checkout,      │  │  (compile,   │  │  (unit, lint,     │  │
│   │   fetch deps)    │  │   package)   │  │   integration)    │  │
│   └────────┬────────┘  └──────┬───────┘  └────────┬──────────┘  │
│            │                  │                    │              │
│   ┌────────▼──────────────────▼────────────────────▼──────────┐  │
│   │                  Artifact Registry                         │  │
│   │  (container images, packages, binaries, test reports)      │  │
│   └────────┬──────────────────────────────────────────────────┘  │
└────────────│────────────────────────────────────────────────────┘
             │
┌────────────▼────────────────────────────────────────────────────┐
│                  Continuous Delivery / Deployment (CD)            │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│   │     Dev      │  │   Staging    │  │  Production  │          │
│   │  Environment │─▶│  Environment │─▶│  Environment │          │
│   │  (auto-      │  │  (auto-      │  │  (manual     │          │
│   │   deploy)    │  │   deploy)    │  │   approval)  │          │
│   └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │  Release Strategies: Blue-Green │ Canary │ Rolling       │    │
│   └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
             │
┌────────────▼────────────────────────────────────────────────────┐
│                       CI/CD Platforms                             │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│   │   GitHub     │  │    Azure     │  │   Jenkins    │          │
│   │   Actions    │  │   DevOps     │  │              │          │
│   └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

## 🔑 Key Concepts

```
CI/CD Fundamentals
──────────────────
Continuous Integration  → Automatically build and test code on every change
Continuous Delivery     → Keep code in a deployable state with automated pipelines
Continuous Deployment   → Automatically deploy every change that passes the pipeline
Pipeline                → Sequence of stages (build, test, deploy) triggered by events

Pipeline Components
───────────────────
Trigger   → Event that starts a pipeline (push, PR, schedule, manual)
Stage     → Logical grouping of related jobs (e.g., build, test, deploy)
Job       → Unit of work that runs on a single runner or agent
Step      → Individual command or action within a job
Artifact  → Output of a pipeline stage (binary, image, report)

Deployment Strategies
─────────────────────
Blue-Green  → Two identical environments; switch traffic after validation
Canary      → Gradually shift traffic to the new version
Rolling     → Replace instances one at a time with zero downtime
Feature Flag → Deploy code behind toggles, decouple deploy from release

GitOps Principles
─────────────────
Declarative   → Desired state is described in Git, not applied imperatively
Versioned     → All changes go through Git with full audit history
Automated     → Agents (ArgoCD, Flux) reconcile cluster state with Git
Observable    → Drift detection and alerts when actual state diverges
```

## 📋 Topics Covered

- **Foundations** — CI/CD fundamentals, pipeline anatomy, DevOps culture, feedback loops
- **Continuous Integration** — Build automation, testing in CI, artifact management, caching
- **Continuous Delivery** — Deployment pipelines, environment promotion, release strategies
- **GitHub Actions** — Workflows, actions marketplace, runners, reusable workflows, matrix builds
- **Azure DevOps** — Azure Pipelines, YAML pipelines, service connections, variable groups
- **Jenkins** — Jenkinsfile, shared libraries, pipeline as code, distributed builds
- **GitOps** — ArgoCD, Flux, pull-based deployments, declarative configuration, drift detection
- **Testing in Pipelines** — Unit/integration/E2E tests, test parallelization, quality gates
- **Security in Pipelines** — SAST, DAST, dependency scanning, supply chain security, secrets
- **Best Practices** — Pipeline patterns, trunk-based development, feature flags, monorepo strategies
- **Anti-Patterns** — Common CI/CD mistakes in pipeline design, testing, and deployment

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to CI/CD?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md) then follow [LEARNING-PATH.md](LEARNING-PATH.md)

**Already familiar with CI/CD?** → Jump to [06-GITOPS.md](06-GITOPS.md) or [08-SECURITY-IN-PIPELINES.md](08-SECURITY-IN-PIPELINES.md)

**Going to production?** → Review [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) and [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [LEARNING-PATH.md](LEARNING-PATH.md) — progressive exercises from basics to production
