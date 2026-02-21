# Infrastructure as Code Learning Resources

A comprehensive guide to Infrastructure as Code (IaC) — from foundational concepts and declarative configuration to Terraform, Pulumi, CloudFormation, state management, multi-cloud patterns, and production-ready IaC workflows.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | IaC fundamentals, declarative vs imperative, state management | **Start here** |
| [01-TERRAFORM](01-TERRAFORM.md) | HCL, providers, modules, state backends | When using Terraform |
| [02-PULUMI](02-PULUMI.md) | General-purpose languages for IaC, stacks, automation API | When using Pulumi |
| [03-CLOUDFORMATION](03-CLOUDFORMATION.md) | AWS-native IaC, nested stacks, drift detection | When using AWS CloudFormation |
| [04-STATE-MANAGEMENT](04-STATE-MANAGEMENT.md) | Remote state, locking, import, state surgery | When managing IaC state |
| [05-MODULES-AND-REUSE](05-MODULES-AND-REUSE.md) | Module design, versioning, registries | When building reusable IaC |
| [06-TESTING](06-TESTING.md) | Policy as code, plan validation, integration testing | When testing infrastructure |
| [07-SECRETS-AND-VARIABLES](07-SECRETS-AND-VARIABLES.md) | Sensitive values, variable hierarchies, environments | **Essential — secrets management** |
| [08-MULTI-CLOUD](08-MULTI-CLOUD.md) | Multi-cloud patterns, abstraction layers | When targeting multiple clouds |
| [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Directory structure, naming, CI/CD integration | **Essential — production checklist** |
| [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | Common IaC mistakes and how to avoid them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured learning guide with exercises | **Start here** after the Overview |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand what Infrastructure as Code is and why it matters
   - Learn the difference between declarative and imperative approaches
   - Explore the IaC lifecycle: write, plan, apply, destroy

2. **Learn Terraform** ([01-TERRAFORM](01-TERRAFORM.md))
   - Write HCL configurations for cloud resources
   - Understand providers, resources, and data sources
   - Manage state with remote backends

3. **Understand State Management** ([04-STATE-MANAGEMENT](04-STATE-MANAGEMENT.md))
   - Remote state storage and locking
   - State import and migration
   - Workspace strategies for multiple environments

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured curriculum with hands-on exercises
   - Progressive skill building from basics to production

### For Experienced Users

1. **Review Best Practices** ([09-BEST-PRACTICES](09-BEST-PRACTICES.md))
   - Production-ready directory structures and naming conventions
   - CI/CD integration patterns for IaC workflows
   - Code review and approval processes for infrastructure changes

2. **Avoid Anti-Patterns** ([10-ANTI-PATTERNS](10-ANTI-PATTERNS.md))
   - Common IaC mistakes and misconfigurations
   - State management pitfalls and how to recover
   - Security and compliance failures

3. **Build Reusable Modules** ([05-MODULES-AND-REUSE](05-MODULES-AND-REUSE.md))
   - Module design patterns and composition
   - Versioning strategies and module registries
   - Cross-team module sharing

4. **Multi-Cloud Strategies** ([08-MULTI-CLOUD](08-MULTI-CLOUD.md))
   - Abstraction layers and provider-agnostic patterns
   - Multi-cloud networking and identity
   - When multi-cloud makes sense (and when it does not)

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                       Developer Workflow                          │
│   Write HCL/Code │ git commit │ pull request │ code review       │
└───────────────────────────┬─────────────────────────────────────┘
                            │ Trigger (push / PR / manual)
┌───────────────────────────▼─────────────────────────────────────┐
│                    IaC CI/CD Pipeline                             │
│   ┌─────────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│   │   Validate       │  │   Plan       │  │   Policy Check    │  │
│   │  (fmt, lint,     │  │  (terraform  │  │  (OPA, Sentinel,  │  │
│   │   validate)      │  │   plan)      │  │   Checkov)        │  │
│   └────────┬────────┘  └──────┬───────┘  └────────┬──────────┘  │
│            │                  │                    │              │
│   ┌────────▼──────────────────▼────────────────────▼──────────┐  │
│   │                    Plan Artifact                           │  │
│   │  (saved plan file, cost estimate, compliance report)       │  │
│   └────────┬──────────────────────────────────────────────────┘  │
└────────────│────────────────────────────────────────────────────┘
             │ Manual Approval
┌────────────▼────────────────────────────────────────────────────┐
│                         Apply Phase                              │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│   │     Dev      │  │   Staging    │  │  Production  │          │
│   │  Environment │─▶│  Environment │─▶│  Environment │          │
│   │  (auto-      │  │  (auto-      │  │  (manual     │          │
│   │   apply)     │  │   apply)     │  │   approval)  │          │
│   └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
             │
┌────────────▼────────────────────────────────────────────────────┐
│                       State Management                           │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│   │  Remote      │  │   State      │  │   Drift      │          │
│   │  Backend     │  │   Locking    │  │   Detection  │          │
│   │  (S3, GCS,   │  │  (DynamoDB,  │  │  (reconcile  │          │
│   │   Azure Blob)│  │   Consul)    │  │   or alert)  │          │
│   └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

## 🔑 Key Concepts

```
IaC Fundamentals
────────────────
Declarative    → Describe the desired end state; the tool figures out how to get there
Imperative     → Describe the exact steps to execute in order
Idempotent     → Running the same configuration twice produces the same result
State          → A record of what infrastructure exists and how it maps to configuration

IaC Lifecycle
─────────────
Write   → Define infrastructure in code (HCL, TypeScript, YAML, Python)
Plan    → Preview what changes will be made before applying
Apply   → Execute the planned changes against the cloud provider
Destroy → Tear down all resources defined in the configuration

IaC Tools
─────────
Terraform       → HashiCorp's declarative IaC tool using HCL; multi-cloud
OpenTofu        → Open-source fork of Terraform under the Linux Foundation
Pulumi          → IaC using general-purpose languages (TypeScript, Python, Go, C#)
CloudFormation  → AWS-native declarative IaC using JSON/YAML templates
Bicep           → Azure-native DSL that compiles to ARM templates
CDK             → AWS Cloud Development Kit; imperative code that generates CloudFormation
Crossplane      → Kubernetes-native IaC using CRDs and controllers

State Management
────────────────
Remote Backend  → Store state in a shared, durable location (S3, GCS, Azure Blob)
State Locking   → Prevent concurrent modifications to the same state
State Import    → Bring existing resources under IaC management
Drift Detection → Detect when actual infrastructure diverges from declared state
```

## 📋 Topics Covered

- **Foundations** — IaC fundamentals, declarative vs imperative, idempotency, state management
- **Terraform** — HCL, providers, resources, modules, state backends, Terraform Cloud
- **Pulumi** — General-purpose languages for IaC, stacks, automation API, component resources
- **CloudFormation** — AWS-native IaC, nested stacks, change sets, drift detection, CDK
- **State Management** — Remote state, locking, workspaces, import, state surgery, migration
- **Modules & Reuse** — Module design patterns, versioning, registries, cross-team sharing
- **Testing** — Policy as code (OPA, Sentinel), plan validation, integration testing, Checkov
- **Secrets & Variables** — Sensitive values, variable hierarchies, environment-specific config
- **Multi-Cloud** — Provider-agnostic patterns, abstraction layers, multi-cloud networking
- **Best Practices** — Directory structure, naming conventions, CI/CD integration, code review
- **Anti-Patterns** — Common IaC mistakes in state management, security, and workflow design

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to IaC?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md) then follow [LEARNING-PATH.md](LEARNING-PATH.md)

**Already familiar with IaC?** → Jump to [05-MODULES-AND-REUSE.md](05-MODULES-AND-REUSE.md) or [08-MULTI-CLOUD.md](08-MULTI-CLOUD.md)

**Going to production?** → Review [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) and [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [LEARNING-PATH.md](LEARNING-PATH.md) — progressive exercises from basics to production
