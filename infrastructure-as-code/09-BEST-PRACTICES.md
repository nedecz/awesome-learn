# Infrastructure as Code Best Practices

A comprehensive guide to Infrastructure as Code best practices for production — covering directory structure, naming conventions, code organization, version pinning, CI/CD integration, code review, state management, environment management, tagging, documentation, drift management, blast radius reduction, and operational readiness checklists.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Directory Structure](#directory-structure)
   - [Monorepo vs Polyrepo](#monorepo-vs-polyrepo)
   - [Terraform Project Layout](#terraform-project-layout)
   - [Pulumi Project Layout](#pulumi-project-layout)
   - [Layered Architecture](#layered-architecture)
3. [Naming Conventions](#naming-conventions)
   - [Resource Naming](#resource-naming)
   - [Variable Naming](#variable-naming)
   - [File Naming](#file-naming)
   - [Module Naming](#module-naming)
   - [Tag Naming](#tag-naming)
4. [Code Organization](#code-organization)
   - [File Granularity](#file-granularity)
   - [Data Sources and Locals](#data-sources-and-locals)
   - [Output Organization](#output-organization)
5. [Version Pinning](#version-pinning)
   - [Provider Version Constraints](#provider-version-constraints)
   - [Module Version Constraints](#module-version-constraints)
   - [Terraform Version Constraints](#terraform-version-constraints)
   - [Lock Files](#lock-files)
6. [CI/CD Integration](#cicd-integration)
   - [Plan on PR](#plan-on-pr)
   - [Apply on Merge](#apply-on-merge)
   - [Approval Workflows](#approval-workflows)
   - [Automated Formatting and Validation](#automated-formatting-and-validation)
7. [Code Review for Infrastructure](#code-review-for-infrastructure)
   - [What Reviewers Should Look For](#what-reviewers-should-look-for)
   - [Plan Output Review](#plan-output-review)
   - [Security Review](#security-review)
   - [Cost Review](#cost-review)
   - [Blast Radius Assessment](#blast-radius-assessment)
8. [State Management Best Practices](#state-management-best-practices)
   - [One State per Boundary](#one-state-per-boundary)
   - [Remote Backends](#remote-backends)
   - [State Locking](#state-locking)
   - [State Access Control](#state-access-control)
   - [State Encryption](#state-encryption)
9. [Environment Management](#environment-management)
   - [Workspace Strategies](#workspace-strategies)
   - [Directory per Environment](#directory-per-environment)
   - [Tfvars per Environment](#tfvars-per-environment)
   - [Stack per Environment in Pulumi](#stack-per-environment-in-pulumi)
10. [Tagging and Labeling](#tagging-and-labeling)
    - [Mandatory Tags](#mandatory-tags)
    - [Tag Enforcement](#tag-enforcement)
    - [Tag-Based Policies](#tag-based-policies)
11. [Documentation](#documentation)
    - [README per Module](#readme-per-module)
    - [Input and Output Documentation](#input-and-output-documentation)
    - [Runbooks](#runbooks)
    - [Architecture Decision Records](#architecture-decision-records)
12. [Drift Management](#drift-management)
    - [Scheduled Drift Detection](#scheduled-drift-detection)
    - [Alerting](#alerting)
    - [Remediation Workflows](#remediation-workflows)
13. [Blast Radius Reduction](#blast-radius-reduction)
    - [Small State Files](#small-state-files)
    - [Targeted Applies](#targeted-applies)
    - [State Splitting Strategies](#state-splitting-strategies)
14. [IaC Production Checklist](#iac-production-checklist)
15. [Next Steps](#next-steps)
16. [Version History](#version-history)

---

## Overview

Infrastructure as Code transforms cloud resource provisioning from a manual, error-prone process into a repeatable, auditable, and version-controlled discipline. But writing IaC that works once is not the same as writing IaC that scales across teams, environments, and cloud providers. Without conventions, structure, and automation, IaC repositories devolve into tangled configurations that nobody dares to touch.

This document distills IaC best practices from production operations, platform engineering teams, and multi-cloud environments into actionable guidance for teams building and managing infrastructure through code.

### Target Audience

- **DevOps Engineers** designing and maintaining IaC repositories, modules, and pipelines
- **Platform Engineers** defining IaC standards, module libraries, and developer experience for organizations
- **SREs** ensuring infrastructure reliability, drift detection, and blast radius control
- **Developers** provisioning application infrastructure and contributing to shared IaC codebases

### Scope

- Directory structure and project layout for Terraform and Pulumi
- Naming conventions across resources, variables, files, and tags
- Code organization, version pinning, and dependency management
- CI/CD integration with plan-on-PR and apply-on-merge workflows
- State management, environment management, and tagging strategies
- Documentation, drift management, and blast radius reduction

---

## Directory Structure

### Monorepo vs Polyrepo

The first structural decision is whether to keep all IaC in a single repository or split it across multiple repositories.

```
Monorepo vs Polyrepo Decision
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Monorepo                          Polyrepo
  ────────                          ────────
  ┌─────────────────────┐           ┌──────────┐  ┌──────────┐
  │  infra/              │           │ infra-   │  │ infra-   │
  │  ├── networking/     │           │ network  │  │ compute  │
  │  ├── compute/        │           └──────────┘  └──────────┘
  │  ├── database/       │           ┌──────────┐  ┌──────────┐
  │  └── modules/        │           │ infra-   │  │ infra-   │
  └─────────────────────┘           │ database │  │ modules  │
                                    └──────────┘  └──────────┘
  ✅ Single source of truth         ✅ Independent deploy cycles
  ✅ Atomic cross-stack changes     ✅ Fine-grained access control
  ✅ Easier module sharing          ✅ Smaller blast radius per repo
  ⚠️  Large state files              ⚠️  Cross-repo dependencies
  ⚠️  Broad permissions needed       ⚠️  Module versioning overhead
```

| Factor | Monorepo | Polyrepo |
|---|---|---|
| Team size | Small to medium (< 10 contributors) | Large (10+ contributors, multiple teams) |
| Deploy cadence | Coordinated releases | Independent service deploys |
| Access control | Broad — everyone sees everything | Fine-grained per repository |
| Module sharing | Local paths — simple | Git tags or registry — versioned |
| CI/CD complexity | Path-based triggers needed | Simpler per-repo pipelines |
| Cross-cutting changes | Single PR across all stacks | Multiple PRs across repos |

**Recommendation:** Start with a monorepo. Split into polyrepo only when team size, access control, or blast radius demands it.

### Terraform Project Layout

A well-structured Terraform project separates root modules (deployable units) from child modules (reusable components) and environments (configuration variants).

```
terraform/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── terraform.tfvars
│   │   ├── providers.tf
│   │   └── backend.tf
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── terraform.tfvars
│   │   ├── providers.tf
│   │   └── backend.tf
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── terraform.tfvars
│       ├── providers.tf
│       └── backend.tf
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   └── database/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── README.md
└── README.md
```

Each environment directory is a **root module** — it calls child modules, sets variables, and has its own state file and backend configuration. Modules contain no environment-specific configuration.

### Pulumi Project Layout

Pulumi organizes infrastructure as standard programming projects with stacks representing environments.

```
pulumi-infra/
├── Pulumi.yaml                  # Project definition
├── Pulumi.dev.yaml              # Dev stack config
├── Pulumi.staging.yaml          # Staging stack config
├── Pulumi.prod.yaml             # Prod stack config
├── index.ts                     # Entry point
├── networking/
│   ├── vpc.ts
│   └── index.ts
├── compute/
│   ├── ecs.ts
│   └── index.ts
├── database/
│   ├── rds.ts
│   └── index.ts
├── package.json
└── tsconfig.json
```

### Layered Architecture

Large infrastructures benefit from a layered approach where lower layers provide outputs consumed by higher layers. Each layer has its own state and deploy lifecycle.

```
Layered Infrastructure Architecture
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Layer 3: Application          ← Deploys frequently (daily)
  ┌─────────────────────────┐
  │  ECS Services, Lambda,  │
  │  API Gateway, CDN       │
  └────────────┬────────────┘
               │ reads outputs from
  Layer 2: Data                 ← Deploys occasionally (weekly)
  ┌─────────────────────────┐
  │  RDS, ElastiCache,      │
  │  S3 Buckets, DynamoDB   │
  └────────────┬────────────┘
               │ reads outputs from
  Layer 1: Foundation           ← Deploys rarely (monthly)
  ┌─────────────────────────┐
  │  VPC, Subnets, IAM,     │
  │  DNS Zones, KMS Keys    │
  └─────────────────────────┘
```

```hcl
# Layer 2 reads outputs from Layer 1 via remote state
data "terraform_remote_state" "foundation" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "foundation/prod/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_db_subnet_group" "main" {
  name       = "main-db-subnet-group"
  subnet_ids = data.terraform_remote_state.foundation.outputs.private_subnet_ids
}
```

| Layer | Contents | Change Frequency | Blast Radius |
|---|---|---|---|
| Foundation | VPC, IAM, DNS, KMS | Monthly | Very high — everything depends on it |
| Data | Databases, caches, storage | Weekly | High — data loss risk |
| Application | Compute, load balancers, CDN | Daily | Medium — can be redeployed |
| Monitoring | Dashboards, alerts, log groups | As needed | Low — observability only |

---

## Naming Conventions

Consistent naming makes infrastructure readable, searchable, and automatable. Establish a naming convention early and enforce it through code review and policy.

### Resource Naming

Use a consistent pattern for cloud resource names. A common pattern is:

```
{project}-{environment}-{component}-{qualifier}
```

```hcl
# Good — consistent, descriptive, environment-aware
resource "aws_s3_bucket" "logs" {
  bucket = "myapp-prod-logs-us-east-1"
}

resource "aws_rds_instance" "main" {
  identifier = "myapp-prod-postgres-main"
}

resource "aws_ecs_cluster" "main" {
  name = "myapp-prod-ecs-main"
}

# Bad — inconsistent, unclear, no environment
resource "aws_s3_bucket" "bucket1" {
  bucket = "my-bucket"
}

resource "aws_rds_instance" "db" {
  identifier = "database"
}
```

| Component | Pattern | Example |
|---|---|---|
| S3 bucket | `{project}-{env}-{purpose}-{region}` | `myapp-prod-logs-us-east-1` |
| RDS instance | `{project}-{env}-{engine}-{role}` | `myapp-prod-postgres-main` |
| ECS cluster | `{project}-{env}-{service}-{role}` | `myapp-prod-ecs-main` |
| VPC | `{project}-{env}-vpc` | `myapp-prod-vpc` |
| Subnet | `{project}-{env}-{tier}-{az}` | `myapp-prod-private-us-east-1a` |
| Security group | `{project}-{env}-{service}-sg` | `myapp-prod-api-sg` |
| IAM role | `{project}-{env}-{service}-role` | `myapp-prod-api-role` |

### Variable Naming

Use `snake_case` for all Terraform variables. Names should be descriptive and self-documenting.

```hcl
# Good — descriptive, typed, validated
variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "database_instance_class" {
  description = "RDS instance class for the primary database"
  type        = string
  default     = "db.t3.medium"
}

variable "enable_deletion_protection" {
  description = "Whether to enable deletion protection on critical resources"
  type        = bool
  default     = true
}

# Bad — vague, untyped, no description
variable "env" {}
variable "size" {}
variable "flag" {}
```

### File Naming

Use consistent file names across all modules and root configurations.

| File | Purpose |
|---|---|
| `main.tf` | Primary resource definitions and module calls |
| `variables.tf` | Input variable declarations |
| `outputs.tf` | Output value declarations |
| `providers.tf` | Provider configuration and required_providers |
| `backend.tf` | Backend configuration (root modules only) |
| `data.tf` | Data source definitions |
| `locals.tf` | Local value definitions |
| `versions.tf` | Terraform and provider version constraints |

### Module Naming

Name modules after what they create, not how they create it.

```hcl
# Good — named for what it creates
module "vpc" {
  source = "../../modules/networking"
  # ...
}

module "api_database" {
  source = "../../modules/database"
  # ...
}

# Bad — named for implementation details
module "aws_stuff" {
  source = "../../modules/misc"
  # ...
}
```

### Tag Naming

Use consistent tag key formats. Choose one convention and enforce it.

| Convention | Example Key | When to Use |
|---|---|---|
| PascalCase | `Environment`, `CostCenter` | AWS-native convention |
| kebab-case | `environment`, `cost-center` | GCP / Kubernetes labels |
| snake_case | `environment`, `cost_center` | Internal tooling integration |

```hcl
# Consistent PascalCase tag keys (AWS convention)
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    Team        = var.team_name
    CostCenter  = var.cost_center
    ManagedBy   = "terraform"
    Repository  = var.repository_url
  }
}

resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-${var.environment}-app"
    Role = "application-server"
  })
}
```

---

## Code Organization

### File Granularity

Two main approaches exist for organizing resources across files: one resource per file, or grouped by concern. Choose the approach that matches your team's cognitive load.

```
One Resource per File              Grouped by Concern
━━━━━━━━━━━━━━━━━━━━              ━━━━━━━━━━━━━━━━━━
networking/                        networking/
├── vpc.tf                         ├── main.tf        (VPC + subnets +
├── subnets.tf                     │                   route tables +
├── route_tables.tf                │                   NAT + IGW)
├── nat_gateway.tf                 ├── security.tf    (security groups +
├── internet_gateway.tf            │                   NACLs)
├── security_groups.tf             ├── variables.tf
├── variables.tf                   └── outputs.tf
└── outputs.tf

✅ Easy to find one resource        ✅ Fewer files to navigate
✅ Minimal merge conflicts           ✅ Related resources together
⚠️  Many small files                 ⚠️  Larger files
⚠️  Context switching between files  ⚠️  More merge conflicts
```

**Recommendation:** Group by concern for modules with fewer than 20 resources. Split into per-resource files when a module grows beyond that.

### Data Sources and Locals

Separate data sources and locals into dedicated files to keep `main.tf` focused on resource definitions.

```hcl
# data.tf — all data source lookups in one place
data "aws_caller_identity" "current" {}

data "aws_region" "current" {}

data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

```hcl
# locals.tf — computed values and shared expressions
locals {
  account_id = data.aws_caller_identity.current.account_id
  region     = data.aws_region.current.name
  azs        = slice(data.aws_availability_zones.available.names, 0, 3)

  name_prefix = "${var.project}-${var.environment}"

  common_tags = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
  }
}
```

### Output Organization

Outputs should be well-documented and organized for consumption by other modules or layers.

```hcl
# outputs.tf — grouped by resource type
# VPC outputs
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_cidr_block" {
  description = "The CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

# Subnet outputs
output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

# Security group outputs
output "default_security_group_id" {
  description = "The ID of the default security group"
  value       = aws_security_group.default.id
}
```

---

## Version Pinning

Version pinning prevents unexpected breaking changes from providers, modules, and Terraform itself. Pin everything. Drift in tool versions causes the most insidious and hard-to-debug failures.

### Provider Version Constraints

```hcl
# providers.tf — pin providers with pessimistic constraint
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.31"   # Allows 5.31.x but not 5.32.0
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.85"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.10"
    }
  }
}
```

| Constraint | Meaning | Example | Risk Level |
|---|---|---|---|
| `= 5.31.0` | Exact version | Only 5.31.0 | 🟢 Lowest — no surprises |
| `~> 5.31` | Pessimistic — minor only | 5.31.0 to 5.31.x | 🟢 Low — patch updates only |
| `~> 5.0` | Pessimistic — major only | 5.0.0 to 5.x.x | 🟡 Medium — minor updates |
| `>= 5.0` | Minimum version | 5.0.0 and above | 🔴 High — any future version |
| No constraint | Any version | Whatever is latest | 🔴 Critical — completely unpinned |

### Module Version Constraints

```hcl
# Pin modules from the Terraform Registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.4"

  name = "${local.name_prefix}-vpc"
  cidr = var.vpc_cidr
}

# Pin modules from Git with tags
module "custom_networking" {
  source = "git::https://github.com/myorg/terraform-modules.git//networking?ref=v2.3.1"

  environment = var.environment
}

# Pin modules from Git with commit SHA (most precise)
module "critical_module" {
  source = "git::https://github.com/myorg/terraform-modules.git//security?ref=abc1234def5678"

  environment = var.environment
}
```

### Terraform Version Constraints

```hcl
# versions.tf — pin Terraform version
terraform {
  required_version = "~> 1.7"
}
```

Always specify `required_version` in every root module. Without it, a team member running a different Terraform version can corrupt state or introduce incompatible syntax.

### Lock Files

The `.terraform.lock.hcl` file records the exact provider versions and hashes used. **Always commit this file to version control.**

```hcl
# .terraform.lock.hcl — auto-generated, committed to Git
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.31"
  hashes = [
    "h1:XXXXXXXXXXXXXXXXXXXXXXXXXXXXX=",
    "zh:YYYYYYYYYYYYYYYYYYYYYYYYYYYYY=",
  ]
}
```

```bash
# Update lock file when changing provider versions
terraform init -upgrade

# Verify lock file is committed
git add .terraform.lock.hcl
git commit -m "chore: update provider lock file"
```

| Practice | Recommendation |
|---|---|
| Commit `.terraform.lock.hcl` | ✅ Always — ensures reproducible builds |
| Run `terraform init -upgrade` | Only when intentionally updating providers |
| Pin Terraform version in CI | ✅ Always — use `tfenv` or `asdf` |
| Review lock file changes in PRs | ✅ Always — unexpected changes indicate drift |

---

## CI/CD Integration

Infrastructure changes must flow through the same CI/CD rigor as application code. The standard pattern is: **plan on PR, review the plan, apply on merge**.

### Plan on PR

Every pull request that modifies IaC should automatically run `terraform plan` and post the output as a PR comment. This gives reviewers visibility into exactly what will change.

```yaml
# .github/workflows/terraform-plan.yml
name: Terraform Plan

on:
  pull_request:
    paths:
      - 'terraform/**'

permissions:
  contents: read
  pull-requests: write

jobs:
  plan:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, prod]
    defaults:
      run:
        working-directory: terraform/environments/${{ matrix.environment }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Terraform Init
        run: terraform init -input=false

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        id: plan
        run: terraform plan -input=false -no-color -out=tfplan
        continue-on-error: true

      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan — \`${{ matrix.environment }}\`
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            *Triggered by @${{ github.actor }} in ${{ github.event.pull_request.head.sha }}*`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });
```

### Apply on Merge

After a PR is approved and merged to main, run `terraform apply` automatically or with a manual approval gate.

```yaml
# .github/workflows/terraform-apply.yml
name: Terraform Apply

on:
  push:
    branches: [main]
    paths:
      - 'terraform/**'

permissions:
  contents: read

jobs:
  apply-dev:
    runs-on: ubuntu-latest
    environment: dev
    defaults:
      run:
        working-directory: terraform/environments/dev
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"
      - run: terraform init -input=false
      - run: terraform apply -input=false -auto-approve

  apply-staging:
    needs: apply-dev
    runs-on: ubuntu-latest
    environment: staging
    defaults:
      run:
        working-directory: terraform/environments/staging
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"
      - run: terraform init -input=false
      - run: terraform apply -input=false -auto-approve

  apply-prod:
    needs: apply-staging
    runs-on: ubuntu-latest
    environment:
      name: prod
      url: https://console.aws.amazon.com
    defaults:
      run:
        working-directory: terraform/environments/prod
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"
      - run: terraform init -input=false
      - run: terraform apply -input=false -auto-approve
```

```
CI/CD Pipeline Flow for IaC
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Pull Request                    Merge to Main
  ────────────                    ─────────────
  ┌──────────┐   ┌──────────┐    ┌──────────┐   ┌──────────┐   ┌──────────┐
  │  fmt +   │──▶│  plan    │    │  apply   │──▶│  apply   │──▶│  apply   │
  │ validate │   │ (all env)│    │  (dev)   │   │ (staging)│   │  (prod)  │
  └──────────┘   └──────────┘    └──────────┘   └──────────┘   └──────────┘
       │              │               │              │          🔒 approval
       ▼              ▼               ▼              ▼              gate
  Format errors   Plan posted     Auto-apply    Auto-apply    Manual approval
  block merge     as PR comment   on merge      after dev     required
```

### Approval Workflows

Production applies should require explicit approval. Use GitHub Environments with required reviewers.

| Environment | Auto-Apply | Approval Required | Reviewers |
|---|---|---|---|
| dev | ✅ Yes | ❌ No | — |
| staging | ✅ Yes | ❌ No | — |
| prod | ❌ No | ✅ Yes | Platform team (2 reviewers) |

### Automated Formatting and Validation

Run formatting and validation checks on every PR to catch issues before they reach review.

```yaml
# Pre-commit checks for Terraform
- name: Terraform Format
  run: terraform fmt -check -recursive -diff

- name: Terraform Validate
  run: |
    terraform init -backend=false
    terraform validate

- name: TFLint
  uses: terraform-linters/setup-tflint@v4
  with:
    tflint_version: latest
- run: |
    tflint --init
    tflint --recursive --format compact

- name: Checkov Security Scan
  uses: bridgecrewio/checkov-action@v12
  with:
    directory: terraform/
    framework: terraform
    quiet: true
```

| Check | Purpose | Blocks Merge |
|---|---|---|
| `terraform fmt` | Consistent formatting | ✅ Yes |
| `terraform validate` | Syntax and type checking | ✅ Yes |
| TFLint | Linting rules and best practices | ✅ Yes |
| Checkov / tfsec | Security policy scanning | ✅ Yes |
| `terraform plan` | Preview of changes | ✅ Yes (on failure) |
| Cost estimation (Infracost) | Budget awareness | ⚠️ Warning only |

---

## Code Review for Infrastructure

Infrastructure code review requires a different lens than application code review. Reviewers must assess security, cost, blast radius, and operational impact — not just correctness.

### What Reviewers Should Look For

```
Infrastructure Code Review Checklist
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Correctness        Security           Cost
  ───────────        ────────           ────
  ☐ Plan output      ☐ No public        ☐ Instance sizes
    matches intent     access by         ☐ Storage volumes
  ☐ No resources       default           ☐ Data transfer
    destroyed        ☐ Encryption at    ☐ Reserved vs
    unexpectedly       rest + transit      on-demand
  ☐ Variables have   ☐ IAM least
    descriptions       privilege         Blast Radius
  ☐ Outputs          ☐ No hardcoded     ─────────────
    documented         secrets           ☐ # of resources
                     ☐ Security           changing
                       groups tight      ☐ State file scope
                                         ☐ Rollback plan
```

### Plan Output Review

Always review the `terraform plan` output before approving a PR. Look for unexpected destroys, replacements, and changes to sensitive resources.

```
Plan Symbols and Their Meaning
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  +   Create     — New resource will be created
  -   Destroy    — Existing resource will be deleted
  ~   Update     — Existing resource will be modified in-place
  -/+ Replace    — Resource will be destroyed and recreated
  <=  Read       — Data source will be read
```

| Plan Symbol | Risk Level | Action Required |
|---|---|---|
| `+` Create | 🟢 Low | Verify resource config is correct |
| `~` Update in-place | 🟡 Medium | Verify change is intentional |
| `-/+` Replace | 🟠 High | Verify downtime is acceptable |
| `-` Destroy | 🔴 Critical | Verify this is intentional |

**Red flags in plan output:**

- Unexpected resource destroys (name changes force replacement)
- Security group rule changes that open wide CIDR ranges
- IAM policy changes that grant `*` permissions
- Database instance changes that trigger replacement (data loss risk)
- Changes to resources you didn't intend to modify

### Security Review

```hcl
# ❌ Bad — overly permissive IAM policy
resource "aws_iam_policy" "bad_example" {
  name = "too-permissive"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "*"
      Resource = "*"
    }]
  })
}

# ✅ Good — least-privilege IAM policy
resource "aws_iam_policy" "good_example" {
  name = "s3-read-only"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:ListBucket"]
      Resource = [
        "arn:aws:s3:::myapp-prod-data",
        "arn:aws:s3:::myapp-prod-data/*"
      ]
    }]
  })
}
```

### Cost Review

Integrate cost estimation into your PR workflow so reviewers can assess the financial impact of infrastructure changes.

```yaml
# GitHub Actions — Infracost cost estimation
- name: Infracost Breakdown
  uses: infracost/actions/setup@v3
  with:
    api-key: ${{ secrets.INFRACOST_API_KEY }}

- name: Generate Cost Diff
  run: |
    infracost diff \
      --path=terraform/environments/prod \
      --compare-to=infracost-base.json \
      --format=json \
      --out-file=/tmp/infracost-diff.json

- name: Post Cost Comment
  run: |
    infracost comment github \
      --path=/tmp/infracost-diff.json \
      --repo=${{ github.repository }} \
      --pull-request=${{ github.event.pull_request.number }} \
      --github-token=${{ secrets.GITHUB_TOKEN }}
```

### Blast Radius Assessment

Before approving, assess how many resources are affected and what the rollback path looks like.

| Change Scope | Blast Radius | Review Rigor |
|---|---|---|
| 1-5 resources, single service | 🟢 Low | Standard review |
| 5-20 resources, single domain | 🟡 Medium | Careful review + plan walkthrough |
| 20+ resources or cross-domain | 🟠 High | Senior review + staged rollout |
| Foundation layer (VPC, IAM, DNS) | 🔴 Critical | Multiple reviewers + change window |

---

## State Management Best Practices

Terraform state is the most critical artifact in your IaC workflow. Corrupted or lost state means Terraform loses track of what it manages. Treat state with the same care you treat production databases.

### One State per Boundary

Split state files along natural boundaries: per environment, per service, or per layer. Never put all infrastructure in a single state file.

```
State File Organization
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ❌ Anti-pattern: monolithic state         ✅ Recommended: split state
  ─────────────────────────────             ──────────────────────────

  ┌───────────────────────────┐             ┌──────────┐  ┌──────────┐
  │  ALL infrastructure       │             │ network  │  │ network  │
  │  ALL environments         │             │   dev    │  │   prod   │
  │  500+ resources           │             └──────────┘  └──────────┘
  │  Slow plan, high risk     │             ┌──────────┐  ┌──────────┐
  └───────────────────────────┘             │ database │  │ database │
                                            │   dev    │  │   prod   │
  Plan takes 10+ minutes                    └──────────┘  └──────────┘
  One mistake affects everything            ┌──────────┐  ┌──────────┐
                                            │   app    │  │   app    │
                                            │   dev    │  │   prod   │
                                            └──────────┘  └──────────┘

                                            Fast plans, isolated blast radius
```

### Remote Backends

Always use a remote backend for team environments. Local state files are for learning only.

```hcl
# backend.tf — S3 backend with DynamoDB locking
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "networking/prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

```hcl
# backend.tf — Azure Storage backend
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate"
    container_name       = "tfstate"
    key                  = "networking/prod/terraform.tfstate"
  }
}
```

```hcl
# backend.tf — GCS backend
terraform {
  backend "gcs" {
    bucket = "mycompany-terraform-state"
    prefix = "networking/prod"
  }
}
```

| Backend | State Locking | Encryption | Best For |
|---|---|---|---|
| S3 + DynamoDB | ✅ DynamoDB | ✅ SSE-S3 / SSE-KMS | AWS-native teams |
| Azure Storage | ✅ Blob lease | ✅ Storage encryption | Azure-native teams |
| GCS | ✅ Built-in | ✅ Google-managed keys | GCP-native teams |
| Terraform Cloud | ✅ Built-in | ✅ Built-in | Multi-cloud teams |
| PostgreSQL | ✅ Advisory locks | ⚠️ Manual config | Self-hosted teams |

### State Locking

State locking prevents concurrent modifications that can corrupt state. Never disable locking in production.

```hcl
# DynamoDB table for state locking (provision this separately)
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name      = "terraform-state-lock"
    ManagedBy = "terraform"
  }
}
```

```bash
# If state is locked and you need to break the lock (emergency only)
terraform force-unlock LOCK_ID

# Always investigate why the lock exists before breaking it
```

### State Access Control

Restrict who can read and write state files. State contains sensitive data including resource attributes, connection strings, and sometimes secrets.

| Access Level | Who | Permissions |
|---|---|---|
| Read + Write | CI/CD pipeline service account | Full state management |
| Read-only | Developers for plan/debugging | Cannot modify state |
| No access | Everyone else | State is not publicly accessible |

```hcl
# S3 bucket policy — restrict state access to CI/CD role
resource "aws_s3_bucket_policy" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "AllowCICDAccess"
        Effect    = "Allow"
        Principal = { AWS = var.cicd_role_arn }
        Action    = ["s3:GetObject", "s3:PutObject"]
        Resource  = "${aws_s3_bucket.terraform_state.arn}/*"
      },
      {
        Sid       = "AllowDeveloperReadOnly"
        Effect    = "Allow"
        Principal = { AWS = var.developer_role_arn }
        Action    = ["s3:GetObject"]
        Resource  = "${aws_s3_bucket.terraform_state.arn}/*"
      }
    ]
  })
}
```

### State Encryption

Encrypt state at rest and in transit. State files contain sensitive infrastructure details.

```hcl
# S3 bucket for state with encryption enforced
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state"

  tags = {
    Name      = "terraform-state"
    ManagedBy = "bootstrap"
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

## Environment Management

Managing multiple environments (dev, staging, prod) is one of the core challenges in IaC. The goal: environments should be structurally identical with only configuration differences.

### Workspace Strategies

Terraform workspaces provide lightweight environment separation within a single configuration directory.

```bash
# Create and switch workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod
terraform workspace select prod

# Reference workspace in configuration
# terraform.workspace returns the current workspace name
```

```hcl
# Using workspace to vary configuration
locals {
  environment = terraform.workspace

  instance_type = {
    dev     = "t3.small"
    staging = "t3.medium"
    prod    = "t3.large"
  }[local.environment]

  instance_count = {
    dev     = 1
    staging = 2
    prod    = 3
  }[local.environment]
}

resource "aws_instance" "app" {
  count         = local.instance_count
  ami           = var.ami_id
  instance_type = local.instance_type

  tags = {
    Name        = "app-${local.environment}-${count.index}"
    Environment = local.environment
  }
}
```

| Approach | Pros | Cons |
|---|---|---|
| Workspaces | Minimal code duplication, same backend | Shared state backend, hard to diverge environments |
| Directory per environment | Full isolation, independent state | Code duplication across directories |
| Tfvars per environment | Single codebase, variable-driven | Must be careful with variable scoping |

### Directory per Environment

The most common and safest approach: a separate directory per environment, each with its own state and backend.

```
environments/
├── dev/
│   ├── main.tf           # Calls modules with dev config
│   ├── terraform.tfvars  # Dev-specific values
│   └── backend.tf        # Dev state backend
├── staging/
│   ├── main.tf           # Calls modules with staging config
│   ├── terraform.tfvars  # Staging-specific values
│   └── backend.tf        # Staging state backend
└── prod/
    ├── main.tf           # Calls modules with prod config
    ├── terraform.tfvars  # Prod-specific values
    └── backend.tf        # Prod state backend
```

### Tfvars per Environment

If you prefer a single configuration directory, use separate `.tfvars` files per environment and pass them at runtime.

```hcl
# terraform.tfvars (dev)
environment    = "dev"
instance_type  = "t3.small"
instance_count = 1
enable_deletion_protection = false
```

```hcl
# terraform.tfvars (prod)
environment    = "prod"
instance_type  = "t3.large"
instance_count = 3
enable_deletion_protection = true
```

```bash
# Apply with environment-specific tfvars
terraform plan -var-file="environments/dev.tfvars"
terraform plan -var-file="environments/prod.tfvars"
```

### Stack per Environment in Pulumi

Pulumi uses stacks as the native mechanism for environment separation.

```bash
# Create stacks for each environment
pulumi stack init dev
pulumi stack init staging
pulumi stack init prod

# Set stack-specific configuration
pulumi config set --stack dev  instanceType t3.small
pulumi config set --stack prod instanceType t3.large

# Deploy a specific stack
pulumi up --stack prod
```

```yaml
# Pulumi.prod.yaml — stack configuration
config:
  aws:region: us-east-1
  myproject:environment: prod
  myproject:instanceType: t3.large
  myproject:instanceCount: "3"
  myproject:enableDeletionProtection: "true"
```

---

## Tagging and Labeling

Tags are the metadata layer that powers cost allocation, access control, automation, and compliance. Missing or inconsistent tags lead to unattributed costs, ungovernable resources, and audit failures.

### Mandatory Tags

Define a minimum set of required tags for every resource.

| Tag Key | Purpose | Example Value |
|---|---|---|
| `Environment` | Identifies the deployment environment | `prod`, `staging`, `dev` |
| `Project` | Groups resources by project or application | `myapp`, `platform` |
| `Team` | Identifies the owning team | `backend`, `platform` |
| `CostCenter` | Maps to a billing cost center | `CC-1234` |
| `ManagedBy` | Identifies the provisioning tool | `terraform`, `pulumi` |
| `Repository` | Links to the IaC source repository | `github.com/myorg/infra` |

```hcl
# Centralized tag definition using locals
locals {
  mandatory_tags = {
    Environment = var.environment
    Project     = var.project
    Team        = var.team
    CostCenter  = var.cost_center
    ManagedBy   = "terraform"
    Repository  = "github.com/myorg/infra"
  }
}

# Apply to every resource
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags       = merge(local.mandatory_tags, { Name = "${var.project}-${var.environment}-vpc" })
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = local.azs[count.index]

  tags = merge(local.mandatory_tags, {
    Name = "${var.project}-${var.environment}-private-${local.azs[count.index]}"
    Tier = "private"
  })
}
```

### Tag Enforcement

Use AWS Config rules, Azure Policy, or OPA to enforce tag presence at the cloud provider level.

```hcl
# AWS Config rule — require mandatory tags
resource "aws_config_config_rule" "required_tags" {
  name = "required-tags"

  source {
    owner             = "AWS"
    source_identifier = "REQUIRED_TAGS"
  }

  input_parameters = jsonencode({
    tag1Key   = "Environment"
    tag2Key   = "Project"
    tag3Key   = "Team"
    tag4Key   = "CostCenter"
    tag5Key   = "ManagedBy"
  })
}
```

```hcl
# Terraform variable validation for tags
variable "tags" {
  description = "Resource tags"
  type        = map(string)

  validation {
    condition = alltrue([
      contains(keys(var.tags), "Environment"),
      contains(keys(var.tags), "Project"),
      contains(keys(var.tags), "Team"),
    ])
    error_message = "Tags must include Environment, Project, and Team."
  }
}
```

### Tag-Based Policies

Tags enable powerful policies for cost management, access control, and automation.

```hcl
# IAM policy — restrict actions to resources with a specific tag
resource "aws_iam_policy" "environment_restricted" {
  name = "restrict-to-dev-environment"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["ec2:*"]
        Resource = "*"
        Condition = {
          StringEquals = {
            "aws:ResourceTag/Environment" = "dev"
          }
        }
      }
    ]
  })
}
```

| Policy Type | Tag Used | Effect |
|---|---|---|
| Cost allocation | `CostCenter` | Attribute costs to business units |
| Access control | `Environment` | Restrict dev team to dev resources |
| Auto-shutdown | `Schedule` | Stop non-prod instances after hours |
| Backup policy | `BackupPolicy` | Define backup frequency per resource |
| Compliance | `DataClassification` | Enforce encryption for sensitive data |

---

## Documentation

Infrastructure documentation is chronically neglected. When an on-call engineer needs to understand why a VPC is configured a certain way at 3 AM, they need documentation — not a Slack thread from six months ago.

### README per Module

Every reusable module should have a README that explains what it does, how to use it, and what it depends on.

```markdown
# Networking Module

Creates a VPC with public and private subnets, NAT gateways,
and route tables across multiple availability zones.

## Usage

​```hcl
module "networking" {
  source = "../../modules/networking"

  environment     = "prod"
  vpc_cidr        = "10.0.0.0/16"
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
}
​```

## Requirements

| Name | Version |
|------|---------|
| terraform | ~> 1.7 |
| aws | ~> 5.31 |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
| environment | Deployment environment | string | — | yes |
| vpc_cidr | CIDR block for the VPC | string | — | yes |
| private_subnets | List of private subnet CIDRs | list(string) | — | yes |
| public_subnets | List of public subnet CIDRs | list(string) | — | yes |

## Outputs

| Name | Description |
|------|-------------|
| vpc_id | The ID of the VPC |
| private_subnet_ids | List of private subnet IDs |
| public_subnet_ids | List of public subnet IDs |
```

### Input and Output Documentation

Use `terraform-docs` to auto-generate documentation from your code. Integrate it into your CI pipeline.

```yaml
# .github/workflows/terraform-docs.yml
name: Terraform Docs

on:
  pull_request:
    paths:
      - 'terraform/modules/**'

jobs:
  docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.ref }}

      - name: Render terraform-docs
        uses: terraform-docs/gh-actions@v1
        with:
          working-dir: terraform/modules
          recursive: true
          output-file: README.md
          output-method: inject
          git-push: true
```

```bash
# Generate docs locally
terraform-docs markdown table --output-file README.md ./modules/networking/
```

### Runbooks

Create runbooks for common operational tasks. Store them alongside the infrastructure code they relate to.

```markdown
# Runbook: Database Failover

## When to Use
- Primary RDS instance is unresponsive
- Replication lag exceeds 30 seconds
- AWS reports an AZ outage affecting the primary

## Steps

1. Verify the issue:
   ​```bash
   aws rds describe-db-instances --db-instance-identifier myapp-prod-postgres-main
   ​```

2. Initiate failover:
   ​```bash
   aws rds reboot-db-instance \
     --db-instance-identifier myapp-prod-postgres-main \
     --force-failover
   ​```

3. Monitor failover progress:
   ​```bash
   watch -n 5 aws rds describe-events \
     --source-identifier myapp-prod-postgres-main \
     --source-type db-instance
   ​```

4. Verify application connectivity after failover.

## Rollback
Failover to original AZ by repeating step 2.

## Escalation
If failover fails, page the database team via PagerDuty.
```

### Architecture Decision Records

Document significant infrastructure decisions using ADRs so future team members understand why things are the way they are.

```markdown
# ADR-003: Use S3 Backend for Terraform State

## Status
Accepted

## Context
We need a remote backend for Terraform state that supports
locking, encryption, and versioning. Options considered:
- Terraform Cloud
- S3 + DynamoDB
- Azure Storage

## Decision
Use S3 with DynamoDB locking. Our infrastructure is
AWS-native, and S3 provides the best integration with our
existing IAM policies and KMS encryption.

## Consequences
- State is encrypted at rest with KMS
- State locking prevents concurrent applies
- S3 versioning allows state recovery
- Teams on Azure or GCP must use a different backend
```

---

## Drift Management

Drift occurs when the actual state of cloud resources diverges from what Terraform state and configuration describe. Manual changes through the console, API calls from scripts, and auto-scaling events all cause drift.

### Scheduled Drift Detection

Run `terraform plan` on a schedule to detect drift before it causes problems.

```yaml
# .github/workflows/drift-detection.yml
name: Drift Detection

on:
  schedule:
    - cron: '0 8 * * 1-5'  # Weekdays at 8 AM UTC

permissions:
  contents: read
  issues: write

jobs:
  detect-drift:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, prod]
    defaults:
      run:
        working-directory: terraform/environments/${{ matrix.environment }}

    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Terraform Init
        run: terraform init -input=false

      - name: Detect Drift
        id: drift
        run: |
          terraform plan -input=false -no-color -detailed-exitcode -out=tfplan 2>&1 | tee plan.txt
          echo "exitcode=$?" >> "$GITHUB_OUTPUT"
        continue-on-error: true

      - name: Create Issue on Drift
        if: steps.drift.outputs.exitcode == '2'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('terraform/environments/${{ matrix.environment }}/plan.txt', 'utf8');
            github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `Infrastructure Drift Detected: ${{ matrix.environment }}`,
              body: `Drift detected in **${{ matrix.environment }}** environment.\n\n\`\`\`\n${plan.substring(0, 60000)}\n\`\`\``,
              labels: ['drift', 'infrastructure']
            });
```

### Alerting

```
Drift Detection and Response Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Scheduled Plan (daily)
       │
       ▼
  ┌──────────────┐    No drift    ┌──────────────┐
  │  terraform   │───────────────▶│   All clear  │
  │  plan        │                │   (exit 0)   │
  └──────┬───────┘                └──────────────┘
         │ Drift detected
         │ (exit 2)
         ▼
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │ Create GitHub│───▶│ Notify Slack │───▶│ Team triages │
  │   Issue      │    │   channel    │    │   the drift  │
  └──────────────┘    └──────────────┘    └──────┬───────┘
                                                  │
                                    ┌─────────────┴─────────────┐
                                    ▼                           ▼
                             ┌──────────────┐          ┌──────────────┐
                             │ Update IaC   │          │  Revert      │
                             │ to match     │          │  manual      │
                             │ actual state │          │  change      │
                             └──────────────┘          └──────────────┘
```

| Drift Exit Code | Meaning | Action |
|---|---|---|
| `0` | No changes — infrastructure matches code | No action needed |
| `1` | Error — plan failed | Investigate and fix |
| `2` | Changes detected — drift exists | Triage and remediate |

### Remediation Workflows

When drift is detected, choose a remediation strategy based on the nature of the change.

| Scenario | Remediation | Example |
|---|---|---|
| Manual console change | Revert by running `terraform apply` | Someone opened a security group port |
| Intentional out-of-band change | Import into Terraform with `terraform import` | Emergency hotfix via CLI |
| Auto-scaling event | Ignore — expected drift | ASG scaled from 2 to 5 instances |
| Upstream provider change | Update Terraform config to match | AWS deprecated an instance type |

```bash
# Import an existing resource into Terraform state
terraform import aws_security_group.web sg-0abc123def456789

# After importing, run plan to verify no remaining drift
terraform plan
```

---

## Blast Radius Reduction

Blast radius is the scope of damage when something goes wrong. A single Terraform state file with 500 resources means a misconfigured `terraform apply` can break everything at once. Reduce blast radius by splitting state and limiting the scope of each apply.

### Small State Files

Target fewer than 50 resources per state file. Large state files are slow to plan, slow to apply, and dangerous to operate.

| State Size | Resources | Plan Time | Risk Level |
|---|---|---|---|
| Small | 1-20 | < 30 seconds | 🟢 Low |
| Medium | 20-50 | 30 sec - 2 min | 🟡 Medium |
| Large | 50-200 | 2 - 10 min | 🟠 High |
| Monolith | 200+ | 10+ min | 🔴 Critical |

```
Splitting a Monolith State
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Before                          After
  ──────                          ─────
  ┌───────────────────────┐       ┌─────────────┐
  │  Single state file    │       │  networking  │  15 resources
  │  300 resources        │       └─────────────┘
  │  12 min plan time     │       ┌─────────────┐
  │  One bad apply =      │       │  database    │  10 resources
  │  everything breaks    │       └─────────────┘
  └───────────────────────┘       ┌─────────────┐
                                  │  compute     │  25 resources
                                  └─────────────┘
                                  ┌─────────────┐
                                  │  monitoring  │  20 resources
                                  └─────────────┘

                                  Each state: < 30 sec plan
                                  Isolated blast radius
```

### Targeted Applies

Use `-target` to apply changes to specific resources when you need to limit scope. But use it sparingly — it can leave state inconsistent if overused.

```bash
# Target a specific resource
terraform apply -target=aws_security_group.api

# Target a specific module
terraform apply -target=module.database

# Target multiple resources
terraform apply -target=aws_instance.app -target=aws_eip.app
```

| Practice | Recommendation |
|---|---|
| `-target` for emergency fixes | ✅ Acceptable — document why |
| `-target` for routine deploys | ❌ Anti-pattern — split state instead |
| `-target` in CI/CD pipelines | ❌ Never — always apply full plan |
| `-target` for debugging | ✅ Acceptable — in dev only |

### State Splitting Strategies

When you need to extract resources from a large state into a new, smaller state, use `terraform state mv`.

```bash
# Move resources from monolith state to a new networking state
# Step 1: Move resources in state
terraform state mv \
  'aws_vpc.main' \
  'module.networking.aws_vpc.main'

# Step 2: Move to a completely new state file
terraform state mv \
  -state=terraform.tfstate \
  -state-out=networking/terraform.tfstate \
  'aws_vpc.main'

# Step 3: Verify both states are consistent
terraform plan   # In the original directory — should show no changes
cd networking && terraform plan  # In the new directory — should show no changes
```

Use `moved` blocks for refactoring within the same state (Terraform 1.1+):

```hcl
# Refactoring: rename a resource without destroying it
moved {
  from = aws_instance.server
  to   = aws_instance.app
}

# Refactoring: move a resource into a module
moved {
  from = aws_security_group.api
  to   = module.networking.aws_security_group.api
}
```

---

## IaC Production Checklist

| Category | Check | Priority | Status |
|---|---|---|---|
| **Structure** | Directory layout follows layered architecture | 🟠 High | ☐ |
| **Structure** | Modules separated from root configurations | 🔴 Critical | ☐ |
| **Structure** | One state file per environment per service | 🔴 Critical | ☐ |
| **Structure** | State files contain fewer than 50 resources | 🟠 High | ☐ |
| **Naming** | Resource naming convention documented and enforced | 🟠 High | ☐ |
| **Naming** | Variables use snake_case with descriptions | 🟠 High | ☐ |
| **Naming** | Consistent file naming (main.tf, variables.tf, outputs.tf) | 🟠 High | ☐ |
| **Versioning** | Terraform version pinned with `required_version` | 🔴 Critical | ☐ |
| **Versioning** | Provider versions pinned with pessimistic constraints | 🔴 Critical | ☐ |
| **Versioning** | Module versions pinned (registry or Git tags) | 🔴 Critical | ☐ |
| **Versioning** | `.terraform.lock.hcl` committed to version control | 🔴 Critical | ☐ |
| **CI/CD** | `terraform fmt` check on every PR | 🟠 High | ☐ |
| **CI/CD** | `terraform validate` on every PR | 🟠 High | ☐ |
| **CI/CD** | `terraform plan` posted as PR comment | 🔴 Critical | ☐ |
| **CI/CD** | Apply requires merge to main branch | 🔴 Critical | ☐ |
| **CI/CD** | Production apply requires manual approval | 🔴 Critical | ☐ |
| **CI/CD** | Security scanning (Checkov / tfsec) in pipeline | 🟠 High | ☐ |
| **State** | Remote backend configured (S3, GCS, Azure Storage) | 🔴 Critical | ☐ |
| **State** | State locking enabled | 🔴 Critical | ☐ |
| **State** | State encrypted at rest | 🔴 Critical | ☐ |
| **State** | State access restricted to CI/CD and authorized users | 🟠 High | ☐ |
| **State** | State bucket versioning enabled | 🟠 High | ☐ |
| **Security** | No hardcoded secrets in IaC files | 🔴 Critical | ☐ |
| **Security** | IAM policies follow least privilege | 🔴 Critical | ☐ |
| **Security** | Encryption enabled on all data stores | 🔴 Critical | ☐ |
| **Security** | Public access blocked on storage buckets | 🔴 Critical | ☐ |
| **Tags** | Mandatory tags defined and applied to all resources | 🟠 High | ☐ |
| **Tags** | Tag enforcement via AWS Config / Azure Policy | 🟡 Medium | ☐ |
| **Tags** | Cost allocation tags configured | 🟠 High | ☐ |
| **Drift** | Scheduled drift detection (daily or weekly) | 🟠 High | ☐ |
| **Drift** | Drift alerts sent to team channel | 🟡 Medium | ☐ |
| **Drift** | Drift remediation runbook exists | 🟠 High | ☐ |
| **Docs** | README exists for every reusable module | 🟠 High | ☐ |
| **Docs** | terraform-docs auto-generates input/output docs | 🟡 Medium | ☐ |
| **Docs** | Runbooks exist for common operational tasks | 🟠 High | ☐ |
| **Docs** | ADRs document significant infrastructure decisions | 🟡 Medium | ☐ |

---

## Next Steps

Continue your Infrastructure as Code learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | IaC Fundamentals | Core concepts, declarative vs imperative, IaC benefits and workflow |
| [01-TERRAFORM.md](01-TERRAFORM.md) | Terraform | HCL syntax, providers, resources, state, and CLI commands |
| [02-PULUMI.md](02-PULUMI.md) | Pulumi | General-purpose languages for infrastructure, stacks, and providers |
| [03-CLOUDFORMATION.md](03-CLOUDFORMATION.md) | CloudFormation | AWS-native IaC with templates, stacks, and change sets |
| [04-STATE-MANAGEMENT.md](04-STATE-MANAGEMENT.md) | State Management | Remote backends, locking, import, and state operations |
| [05-MODULES-AND-REUSE.md](05-MODULES-AND-REUSE.md) | Modules and Reuse | Reusable modules, module registries, and composition patterns |
| [06-TESTING.md](06-TESTING.md) | Testing | Unit tests, integration tests, policy-as-code, and validation |
| [07-SECRETS-AND-VARIABLES.md](07-SECRETS-AND-VARIABLES.md) | Secrets and Variables | Variable management, secret injection, and sensitive data handling |
| [08-MULTI-CLOUD.md](08-MULTI-CLOUD.md) | Multi-Cloud | Cross-cloud strategies, provider abstraction, and portability |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial IaC Best Practices documentation |
