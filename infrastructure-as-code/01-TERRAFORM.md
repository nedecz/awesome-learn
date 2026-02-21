# Terraform

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [What is Terraform](#what-is-terraform)
   - [HashiCorp and the Origins of Terraform](#hashicorp-and-the-origins-of-terraform)
   - [HashiCorp Configuration Language (HCL)](#hashicorp-configuration-language-hcl)
   - [Open-Source History and the OpenTofu Fork](#open-source-history-and-the-opentofu-fork)
   - [Terraform vs Other IaC Tools](#terraform-vs-other-iac-tools)
3. [HCL Fundamentals](#hcl-fundamentals)
   - [Resources](#resources)
   - [Data Sources](#data-sources)
   - [Variables](#variables)
   - [Outputs](#outputs)
   - [Locals](#locals)
   - [Expressions and Operators](#expressions-and-operators)
   - [Built-in Functions](#built-in-functions)
4. [Providers](#providers)
   - [Provider Architecture](#provider-architecture)
   - [Provider Configuration](#provider-configuration)
   - [Version Constraints](#version-constraints)
   - [Popular Providers](#popular-providers)
   - [Provider Aliases](#provider-aliases)
5. [Resource Lifecycle](#resource-lifecycle)
   - [Create, Update, and Destroy](#create-update-and-destroy)
   - [Meta-Arguments](#meta-arguments)
   - [depends_on](#depends_on)
   - [count](#count)
   - [for_each](#for_each)
   - [lifecycle Block](#lifecycle-block)
6. [Modules](#modules)
   - [Module Structure](#module-structure)
   - [Input Variables and Output Values](#input-variables-and-output-values)
   - [Module Sources](#module-sources)
   - [Module Composition](#module-composition)
7. [State Management](#state-management)
   - [What is Terraform State](#what-is-terraform-state)
   - [Local State](#local-state)
   - [Remote Backends](#remote-backends)
   - [State Backend Comparison](#state-backend-comparison)
   - [Workspaces](#workspaces)
8. [Terraform CLI](#terraform-cli)
   - [Core Workflow Commands](#core-workflow-commands)
   - [Formatting and Validation](#formatting-and-validation)
   - [State Commands](#state-commands)
   - [Import Command](#import-command)
9. [Terraform Cloud and Enterprise](#terraform-cloud-and-enterprise)
   - [Remote Execution](#remote-execution)
   - [VCS Integration](#vcs-integration)
   - [Policy as Code with Sentinel](#policy-as-code-with-sentinel)
10. [Data Sources and Dynamic Blocks](#data-sources-and-dynamic-blocks)
    - [Data Source Patterns](#data-source-patterns)
    - [Dynamic Blocks](#dynamic-blocks)
11. [Provisioners](#provisioners)
    - [When to Use Provisioners](#when-to-use-provisioners)
    - [Why to Avoid Provisioners](#why-to-avoid-provisioners)
    - [Alternatives to Provisioners](#alternatives-to-provisioners)
12. [Terraform Import and Migration](#terraform-import-and-migration)
    - [Importing Existing Resources](#importing-existing-resources)
    - [Import Blocks (Terraform 1.5+)](#import-blocks-terraform-15)
    - [Migration Strategies](#migration-strategies)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **Terraform** — HashiCorp's infrastructure as code tool that enables teams to define, provision, and manage cloud infrastructure using a declarative configuration language. It covers HCL fundamentals, providers, resource lifecycle, modules, state management, the Terraform CLI, Terraform Cloud and Enterprise features, and practical patterns for production-grade infrastructure management.

### Target Audience

- **Developers** provisioning cloud resources alongside their application code and contributing to Terraform codebases
- **DevOps Engineers** designing and maintaining Terraform configurations, modules, and automation pipelines
- **Platform Engineers** building reusable Terraform modules, self-service infrastructure platforms, and golden paths for development teams
- **Site Reliability Engineers (SREs)** managing infrastructure state, drift remediation, and operational consistency across environments
- **Architects** evaluating Terraform strategies, multi-cloud patterns, and organizational standards for infrastructure management

### Scope

- What Terraform is, its history, and how it compares to other IaC tools
- HCL syntax fundamentals: resources, data sources, variables, outputs, locals, expressions, and functions
- Provider architecture and configuration for AWS, Azure, GCP, and Kubernetes
- Resource lifecycle management and meta-arguments
- Module design, composition, and sourcing strategies
- State management: local state, remote backends, locking, and workspaces
- Terraform CLI commands for the full infrastructure lifecycle
- Terraform Cloud and Enterprise: remote execution, VCS integration, and policy as code

---

## What is Terraform

### HashiCorp and the Origins of Terraform

Terraform was created by **HashiCorp** and first released in **July 2014** by Mitchell Hashimoto and Armon Dadgar. It was designed to solve a fundamental problem: managing infrastructure across multiple cloud providers and services using a single, consistent workflow.

Before Terraform, teams relied on provider-specific tools (AWS CloudFormation, Azure ARM templates) or ad-hoc scripts. Terraform introduced a **provider-based plugin architecture** that could manage resources across any API — from major cloud providers to SaaS platforms, DNS providers, and even internal services.

Key milestones in Terraform's history:

| Year | Milestone |
|---|---|
| 2014 | Terraform 0.1 released — initial open-source release |
| 2017 | Terraform 0.10 — provider/core split, separate provider binaries |
| 2020 | Terraform 0.13 — automatic provider installation, `for_each` on modules |
| 2021 | Terraform 1.0 — stability guarantee, production-ready commitment |
| 2023 | Terraform 1.5 — `import` blocks, `check` blocks |
| 2023 | License change from MPL 2.0 to BSL 1.1 |

### HashiCorp Configuration Language (HCL)

Terraform configurations are written in **HashiCorp Configuration Language (HCL)**, a declarative language designed specifically for infrastructure definitions. HCL strikes a balance between human readability and machine parseability:

```hcl
# HCL is declarative — you describe WHAT you want, not HOW to create it
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}
```

**Why HCL instead of JSON, YAML, or a general-purpose language?**

| Aspect | HCL | JSON/YAML | General-Purpose (Python, TypeScript) |
|---|---|---|---|
| Readability | High — purpose-built for infrastructure | Medium — verbose (JSON) or indentation-sensitive (YAML) | Medium — requires understanding the language |
| Expressiveness | Built-in functions, conditionals, loops | Limited — no logic | Full programming language |
| Learnability | Low barrier — small syntax surface | Low barrier | Higher barrier — must know the language |
| Tooling | `terraform fmt`, `terraform validate` | Generic linters | Language-specific tooling |
| Plan/Preview | Native `terraform plan` | Tool-dependent | Tool-dependent |

### Open-Source History and the OpenTofu Fork

Terraform was originally released under the **Mozilla Public License 2.0 (MPL 2.0)**, a permissive open-source license. In **August 2023**, HashiCorp changed the license to the **Business Source License 1.1 (BSL 1.1)**, which restricts competitive commercial use.

This license change prompted the creation of **OpenTofu** — a community-driven fork of Terraform maintained by the Linux Foundation. OpenTofu forked from Terraform 1.5.x and remains under the MPL 2.0 license.

```
  Terraform Licensing Timeline
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  2014 ──────────── 2023 Aug ──────────── 2023 Sep ──────────── Now
    │                  │                     │                    │
    │  Terraform       │  HashiCorp changes  │  OpenTofu fork     │
    │  released under  │  license to         │  announced under   │
    │  MPL 2.0         │  BSL 1.1            │  MPL 2.0 (Linux    │
    │  (open source)   │                     │  Foundation)       │
    │                  │                     │                    │
    ▼                  ▼                     ▼                    ▼
  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐
  │  Terraform   │  │  Terraform   │  │  Terraform BSL 1.1         │
  │  MPL 2.0     │  │  BSL 1.1     │  │  OpenTofu  MPL 2.0 (fork) │
  └──────────────┘  └──────────────┘  └────────────────────────────┘
```

**Key differences between Terraform and OpenTofu:**

| Aspect | Terraform | OpenTofu |
|---|---|---|
| License | BSL 1.1 | MPL 2.0 |
| Governance | HashiCorp | Linux Foundation |
| Registry | registry.terraform.io | registry.opentofu.org |
| CLI command | `terraform` | `tofu` |
| Compatibility | N/A | Drop-in replacement (from 1.5.x) |
| State encryption | Not built-in | Native state encryption |

> **Best Practice:** For new projects, evaluate both Terraform and OpenTofu. The HCL syntax, provider ecosystem, and workflow patterns covered in this document apply to both tools. The core concepts are identical — only the binary name and registry differ.

### Terraform vs Other IaC Tools

| Feature | Terraform / OpenTofu | AWS CloudFormation | Pulumi | Ansible |
|---|---|---|---|---|
| Language | HCL | JSON/YAML | Python, TypeScript, Go, C# | YAML |
| Approach | Declarative | Declarative | Imperative + Declarative | Imperative |
| Multi-cloud | Yes | AWS only | Yes | Yes (config mgmt) |
| State | Managed by tool | Managed by AWS | Managed by tool | Stateless |
| Plan/Preview | `terraform plan` | Change sets | `pulumi preview` | `--check` mode |
| Maturity | Very high | Very high (AWS) | Growing | Very high |

---

## HCL Fundamentals

### Resources

Resources are the most important element in Terraform. Each resource block describes one or more infrastructure objects — virtual networks, compute instances, DNS records, or higher-level components like database configurations.

```hcl
# Syntax: resource "<PROVIDER>_<TYPE>" "<LOCAL_NAME>" { ... }

# AWS VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "main-vpc"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# AWS Subnet within the VPC
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-1a"
  }
}

# Azure Resource Group
resource "azurerm_resource_group" "example" {
  name     = "rg-example-eastus"
  location = "East US"

  tags = {
    Environment = "production"
    Team        = "platform"
  }
}

# GCP Cloud Storage Bucket
resource "google_storage_bucket" "assets" {
  name          = "my-company-assets-bucket"
  location      = "US"
  force_destroy = false

  versioning {
    enabled = true
  }

  lifecycle_rule {
    condition {
      age = 90
    }
    action {
      type = "Delete"
    }
  }
}

# Kubernetes Namespace
resource "kubernetes_namespace" "app" {
  metadata {
    name = "my-application"

    labels = {
      environment = "production"
      team        = "backend"
    }
  }
}
```

### Data Sources

Data sources allow Terraform to read information from existing infrastructure or external systems. They do not create or manage resources — they query existing state.

```hcl
# Look up an existing AWS AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Use the data source in a resource
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
}

# Look up an existing Azure subscription
data "azurerm_subscription" "current" {}

# Look up existing GCP project
data "google_project" "current" {}

# Look up availability zones
data "aws_availability_zones" "available" {
  state = "available"
}
```

### Variables

Variables parameterize Terraform configurations. They are declared with `variable` blocks and can be set via CLI flags, environment variables, `.tfvars` files, or defaults.

```hcl
# String variable with default
variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "development"

  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment must be development, staging, or production."
  }
}

# Number variable
variable "instance_count" {
  description = "Number of EC2 instances to create"
  type        = number
  default     = 2
}

# Boolean variable
variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = true
}

# List variable
variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# Map variable
variable "instance_tags" {
  description = "Tags to apply to instances"
  type        = map(string)
  default = {
    ManagedBy = "terraform"
    Team      = "platform"
  }
}

# Object variable with nested structure
variable "database_config" {
  description = "Database configuration"
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    allocated_storage = number
    multi_az       = bool
  })
  default = {
    engine            = "postgres"
    engine_version    = "15.4"
    instance_class    = "db.t3.medium"
    allocated_storage = 100
    multi_az          = true
  }
}

# Sensitive variable — value hidden in plan/apply output
variable "db_password" {
  description = "Database master password"
  type        = string
  sensitive   = true
}
```

**Setting variables — order of precedence (lowest to highest):**

| Method | Example | Precedence |
|---|---|---|
| Default value | `default = "dev"` in variable block | Lowest |
| Environment variable | `export TF_VAR_environment="staging"` | ↑ |
| `terraform.tfvars` file | `environment = "staging"` | ↑ |
| `*.auto.tfvars` file | `environment = "staging"` | ↑ |
| `-var-file` flag | `terraform apply -var-file="prod.tfvars"` | ↑ |
| `-var` flag | `terraform apply -var="environment=production"` | Highest |

### Outputs

Outputs expose values from your Terraform configuration. They are useful for passing data between modules, displaying information after `terraform apply`, and enabling remote state data sharing.

```hcl
# Simple output
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

# Output with sensitive value
output "database_endpoint" {
  description = "The RDS endpoint"
  value       = aws_db_instance.main.endpoint
  sensitive   = true
}

# Computed output
output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

# Output referencing multiple resources
output "instance_public_ips" {
  description = "Public IPs of all web instances"
  value = {
    for instance in aws_instance.web :
    instance.tags["Name"] => instance.public_ip
  }
}
```

### Locals

Locals define computed values that can be reused within a module. They reduce repetition and make configurations more readable.

```hcl
locals {
  # Common tags applied to all resources
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "terraform"
    Team        = var.team_name
  }

  # Computed values
  name_prefix  = "${var.project_name}-${var.environment}"
  is_production = var.environment == "production"

  # Conditional logic
  instance_type = local.is_production ? "t3.large" : "t3.micro"
  min_instances = local.is_production ? 3 : 1
}

# Using locals in resources
resource "aws_instance" "web" {
  count         = local.min_instances
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.instance_type

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web-${count.index + 1}"
  })
}
```

### Expressions and Operators

HCL supports a rich set of expressions for building dynamic configurations:

```hcl
locals {
  # Conditional expression
  instance_type = var.environment == "production" ? "t3.large" : "t3.micro"

  # Arithmetic
  total_storage = var.base_storage * var.instance_count

  # String interpolation
  bucket_name = "company-${var.environment}-${var.region}-assets"

  # For expressions — transform lists
  upper_zones = [for z in var.availability_zones : upper(z)]

  # For expressions — transform maps
  tag_map = { for k, v in var.tags : upper(k) => v }

  # For expressions — filter
  large_instances = [
    for i in var.instances : i
    if i.size == "large"
  ]

  # Splat expressions
  all_instance_ids = aws_instance.web[*].id

  # Null coalescing (try)
  region = coalesce(var.override_region, var.default_region)
}
```

### Built-in Functions

Terraform provides a comprehensive standard library of built-in functions:

```hcl
locals {
  # String functions
  lower_env    = lower(var.environment)           # "PROD" -> "prod"
  name_parts   = split("-", var.resource_name)    # "my-app" -> ["my", "app"]
  joined       = join(", ", var.team_members)      # ["a", "b"] -> "a, b"
  trimmed      = trimspace("  hello  ")           # "hello"
  replaced     = replace("hello-world", "-", "_") # "hello_world"

  # Numeric functions
  max_count    = max(var.min_instances, 3)         # returns the larger value
  min_count    = min(var.max_instances, 10)         # returns the smaller value
  abs_value    = abs(-42)                          # 42

  # Collection functions
  flat_list    = flatten([["a", "b"], ["c"]])      # ["a", "b", "c"]
  unique_list  = distinct(["a", "a", "b"])         # ["a", "b"]
  merged_maps  = merge(var.default_tags, var.extra_tags)
  key_list     = keys(var.instance_map)
  value_list   = values(var.instance_map)
  has_key      = contains(keys(var.tags), "Name")
  list_length  = length(var.subnets)

  # Encoding functions
  user_data    = base64encode(file("scripts/init.sh"))
  config_json  = jsonencode({ name = "app", port = 8080 })
  config_yaml  = yamlencode({ name = "app", port = 8080 })

  # Filesystem functions
  script       = file("${path.module}/scripts/bootstrap.sh")
  template     = templatefile("${path.module}/templates/config.tpl", {
    db_host = var.db_host
    db_port = var.db_port
  })

  # Type conversion functions
  stringified  = tostring(42)
  num          = tonumber("42")
  as_list      = tolist(toset(["a", "b", "a"]))   # ["a", "b"]
  as_map       = tomap({ name = "app" })

  # CIDR functions
  subnet_cidrs = cidrsubnet("10.0.0.0/16", 8, 1)  # "10.0.1.0/24"
}
```

---

## Providers

### Provider Architecture

Providers are plugins that Terraform uses to interact with APIs. Each provider adds a set of resource types and data sources that Terraform can manage. Providers are distributed separately from Terraform itself and are downloaded during `terraform init`.

```
  Terraform Provider Architecture
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────────────────────────────────────────────────────────┐
  │                      Terraform Core                             │
  │                                                                 │
  │   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   │
  │   │  Config   │   │  State   │   │  Plan    │   │  Apply   │   │
  │   │  Parser   │   │  Manager │   │  Engine  │   │  Engine  │   │
  │   └──────────┘   └──────────┘   └──────────┘   └──────────┘   │
  │                          │                                      │
  │                          │  gRPC Protocol                       │
  │                          ▼                                      │
  │   ┌─────────────────────────────────────────────────────────┐   │
  │   │                  Provider Plugin Interface               │   │
  │   └─────────────────────────────────────────────────────────┘   │
  └───────────┬───────────┬───────────┬───────────┬─────────────────┘
              │           │           │           │
              ▼           ▼           ▼           ▼
        ┌──────────┐┌──────────┐┌──────────┐┌──────────┐
        │   AWS    ││  Azure   ││   GCP    ││  K8s     │
        │ Provider ││ Provider ││ Provider ││ Provider │
        └────┬─────┘└────┬─────┘└────┬─────┘└────┬─────┘
             │           │           │           │
             ▼           ▼           ▼           ▼
        ┌──────────┐┌──────────┐┌──────────┐┌──────────┐
        │ AWS API  ││Azure API ││ GCP API  ││  K8s API │
        └──────────┘└──────────┘└──────────┘└──────────┘
```

### Provider Configuration

Providers are configured in the `required_providers` block and then set up with provider-specific settings:

```hcl
# terraform block — declare required providers and versions
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

# AWS provider configuration
provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Environment = var.environment
    }
  }
}

# Azure provider configuration
provider "azurerm" {
  features {}

  subscription_id = var.azure_subscription_id
}

# GCP provider configuration
provider "google" {
  project = var.gcp_project_id
  region  = var.gcp_region
}

# Kubernetes provider configuration
provider "kubernetes" {
  host                   = data.aws_eks_cluster.main.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.main.certificate_authority[0].data)
  token                  = data.aws_eks_cluster_auth.main.token
}
```

### Version Constraints

Terraform uses version constraints to control which provider versions are acceptable:

| Constraint | Meaning | Example |
|---|---|---|
| `= 5.0.0` | Exact version | Only version 5.0.0 |
| `>= 5.0.0` | Minimum version | 5.0.0 or newer |
| `~> 5.0` | Pessimistic — allows patch/minor | >= 5.0, < 6.0 |
| `~> 5.0.0` | Pessimistic — allows patch only | >= 5.0.0, < 5.1.0 |
| `>= 5.0, < 6.0` | Range | Between 5.0 and 6.0 |

> **Best Practice:** Use the pessimistic constraint operator (`~>`) to allow patch updates while preventing breaking changes. Lock exact versions in production with `.terraform.lock.hcl` and commit the lock file to version control.

### Popular Providers

| Provider | Source | Primary Use | Resources |
|---|---|---|---|
| AWS | `hashicorp/aws` | Amazon Web Services | EC2, S3, RDS, Lambda, VPC, IAM |
| Azure | `hashicorp/azurerm` | Microsoft Azure | VMs, Storage, AKS, SQL, VNets |
| GCP | `hashicorp/google` | Google Cloud Platform | GCE, GCS, GKE, Cloud SQL, VPC |
| Kubernetes | `hashicorp/kubernetes` | Kubernetes clusters | Deployments, Services, ConfigMaps |
| Helm | `hashicorp/helm` | Helm chart management | Helm releases |
| Docker | `kreuzwerker/docker` | Docker containers | Images, Containers, Networks |
| GitHub | `integrations/github` | GitHub resources | Repos, Teams, Actions secrets |
| Cloudflare | `cloudflare/cloudflare` | DNS and CDN | DNS records, Pages, Workers |
| Datadog | `datadog/datadog` | Monitoring | Monitors, Dashboards, SLOs |
| Random | `hashicorp/random` | Random value generation | Strings, IDs, passwords |

### Provider Aliases

When you need multiple configurations for the same provider (e.g., multi-region deployments):

```hcl
# Default AWS provider — us-east-1
provider "aws" {
  region = "us-east-1"
}

# Aliased AWS provider — eu-west-1
provider "aws" {
  alias  = "europe"
  region = "eu-west-1"
}

# Use the default provider
resource "aws_s3_bucket" "us_bucket" {
  bucket = "my-app-us-data"
}

# Use the aliased provider
resource "aws_s3_bucket" "eu_bucket" {
  provider = aws.europe
  bucket   = "my-app-eu-data"
}
```

---

## Resource Lifecycle

### Create, Update, and Destroy

Terraform manages infrastructure through a predictable lifecycle. When you run `terraform apply`, Terraform compares the desired state (your configuration) with the current state (stored in the state file) and determines the minimal set of changes needed:

```
  Terraform Core Workflow
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌───────────┐     ┌───────────┐     ┌───────────┐
  │           │     │           │     │           │
  │   INIT    │────▶│   PLAN    │────▶│   APPLY   │
  │           │     │           │     │           │
  └───────────┘     └─────┬─────┘     └─────┬─────┘
       │                  │                 │
       │                  │                 │
       ▼                  ▼                 ▼
  ┌───────────┐     ┌───────────┐     ┌───────────┐
  │ Download  │     │ Compare   │     │ Execute   │
  │ providers │     │ config vs │     │ API calls │
  │ and       │     │ state to  │     │ to create,│
  │ modules   │     │ produce   │     │ update, or│
  │           │     │ diff      │     │ destroy   │
  └───────────┘     └───────────┘     └─────┬─────┘
                                            │
                                            ▼
                                      ┌───────────┐
                                      │  Update   │
                                      │  state    │
                                      │  file     │
                                      └───────────┘

  The cycle repeats:
  ┌────────────────────────────────────────────────────────┐
  │  Edit Config ──▶ Plan ──▶ Review ──▶ Apply ──▶ Repeat │
  └────────────────────────────────────────────────────────┘
```

Terraform categorizes changes into three operations:

| Operation | Symbol in Plan | Description |
|---|---|---|
| Create | `+` (green) | Resource does not exist — will be created |
| Update in-place | `~` (yellow) | Resource exists — attributes will be modified |
| Destroy and recreate | `-/+` (red/green) | Change requires replacement (e.g., changing AMI) |
| Destroy | `-` (red) | Resource will be deleted |

### Meta-Arguments

Meta-arguments are special arguments available on every resource block regardless of the resource type.

### depends_on

Explicitly declares dependencies when Terraform cannot automatically infer the ordering:

```hcl
resource "aws_iam_role_policy" "example" {
  name   = "example-policy"
  role   = aws_iam_role.example.id
  policy = data.aws_iam_policy_document.example.json
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  # Ensure the IAM policy is created before the instance
  depends_on = [aws_iam_role_policy.example]
}
```

> **Best Practice:** Avoid `depends_on` when possible. Terraform automatically detects dependencies through resource attribute references. Use `depends_on` only for hidden dependencies that Terraform cannot infer (e.g., IAM policy propagation delays).

### count

Create multiple instances of a resource based on a numeric value:

```hcl
variable "instance_count" {
  type    = number
  default = 3
}

resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  tags = {
    Name = "web-server-${count.index + 1}"
  }
}

# Conditional creation — create resource only in production
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count = var.environment == "production" ? 1 : 0

  alarm_name          = "high-cpu-utilization"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = 80
}
```

### for_each

Create multiple instances based on a map or set, with stable resource addresses:

```hcl
variable "subnets" {
  type = map(object({
    cidr_block        = string
    availability_zone = string
    public            = bool
  }))
  default = {
    "public-1a" = {
      cidr_block        = "10.0.1.0/24"
      availability_zone = "us-east-1a"
      public            = true
    }
    "public-1b" = {
      cidr_block        = "10.0.2.0/24"
      availability_zone = "us-east-1b"
      public            = true
    }
    "private-1a" = {
      cidr_block        = "10.0.10.0/24"
      availability_zone = "us-east-1a"
      public            = false
    }
  }
}

resource "aws_subnet" "this" {
  for_each = var.subnets

  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr_block
  availability_zone       = each.value.availability_zone
  map_public_ip_on_launch = each.value.public

  tags = {
    Name = each.key
    Type = each.value.public ? "public" : "private"
  }
}

# Reference for_each resources
output "subnet_ids" {
  value = { for k, v in aws_subnet.this : k => v.id }
}
```

> **Best Practice:** Prefer `for_each` over `count` for most use cases. With `count`, removing an item from the middle of a list causes all subsequent resources to be destroyed and recreated. With `for_each`, resources are identified by map key, so removing one item only affects that specific resource.

### lifecycle Block

The `lifecycle` block controls how Terraform handles resource creation, update, and destruction:

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  lifecycle {
    # Prevent destruction — useful for databases and stateful resources
    prevent_destroy = true

    # Ignore changes to specific attributes (e.g., tags managed outside Terraform)
    ignore_changes = [
      tags["LastModified"],
      tags["UpdatedBy"],
    ]

    # Create new resource before destroying the old one (blue-green)
    create_before_destroy = true

    # Replace resource when any of these expressions change
    replace_triggered_by = [
      aws_security_group.web.id,
    ]
  }
}
```

| Lifecycle Argument | Use Case |
|---|---|
| `prevent_destroy` | Protect critical resources (databases, S3 buckets) from accidental deletion |
| `ignore_changes` | Attributes managed outside Terraform (auto-scaling, manual tags) |
| `create_before_destroy` | Zero-downtime replacements — new resource is ready before old is removed |
| `replace_triggered_by` | Force replacement when a related resource changes |

---

## Modules

### Module Structure

A Terraform module is a directory containing `.tf` files. Every Terraform configuration is technically a module — the root module. Child modules are called from the root module to encapsulate and reuse infrastructure patterns.

```
  Recommended Module Structure
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  modules/
  └── vpc/
      ├── main.tf          # Resource definitions
      ├── variables.tf     # Input variables
      ├── outputs.tf       # Output values
      ├── versions.tf      # Provider version constraints
      ├── locals.tf        # Local values (optional)
      ├── data.tf          # Data sources (optional)
      └── README.md        # Module documentation
```

```hcl
# modules/vpc/variables.tf
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "public_subnets" {
  description = "List of public subnet CIDR blocks"
  type        = list(string)
}

variable "private_subnets" {
  description = "List of private subnet CIDR blocks"
  type        = list(string)
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
}
```

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

resource "aws_subnet" "public" {
  count = length(var.public_subnets)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnets[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.environment}-public-${var.availability_zones[count.index]}"
    Type = "public"
  }
}

resource "aws_subnet" "private" {
  count = length(var.private_subnets)

  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.environment}-private-${var.availability_zones[count.index]}"
    Type = "private"
  }
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = {
    Name = "${var.environment}-igw"
  }
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.this.id
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "internet_gateway_id" {
  description = "The ID of the Internet Gateway"
  value       = aws_internet_gateway.this.id
}
```

### Input Variables and Output Values

Modules communicate through input variables and output values, creating a clear contract:

```hcl
# Root module — calling the VPC module
module "vpc" {
  source = "./modules/vpc"

  vpc_cidr           = "10.0.0.0/16"
  environment        = "production"
  public_subnets     = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets    = ["10.0.10.0/24", "10.0.11.0/24"]
  availability_zones = ["us-east-1a", "us-east-1b"]
}

# Using module outputs
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  subnet_id     = module.vpc.public_subnet_ids[0]
}

output "vpc_id" {
  value = module.vpc.vpc_id
}
```

### Module Sources

Terraform supports multiple module sources:

```hcl
# Local path
module "vpc" {
  source = "./modules/vpc"
}

# Terraform Registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"
}

# GitHub (HTTPS)
module "vpc" {
  source = "github.com/my-org/terraform-modules//vpc?ref=v1.2.0"
}

# GitHub (SSH)
module "vpc" {
  source = "git@github.com:my-org/terraform-modules.git//vpc?ref=v1.2.0"
}

# S3 bucket
module "vpc" {
  source = "s3::https://my-bucket.s3.amazonaws.com/modules/vpc.zip"
}

# GCS bucket
module "vpc" {
  source = "gcs::https://www.googleapis.com/storage/v1/my-bucket/modules/vpc.zip"
}
```

| Source Type | Use Case | Versioning |
|---|---|---|
| Local path | Development, monorepo | N/A (same repo) |
| Terraform Registry | Public or private shared modules | `version` argument |
| Git (GitHub, GitLab) | Private modules, internal teams | `?ref=` tag/branch/SHA |
| S3/GCS | Air-gapped or custom distribution | Object versioning |

> **Best Practice:** Pin module versions in production. Use semantic versioning for registry modules (`version = "5.1.0"`) and git refs for Git-sourced modules (`?ref=v1.2.0`). Never use `main` or `latest` in production.

### Module Composition

Compose larger systems from smaller, focused modules:

```hcl
# Production infrastructure composed from modules
module "networking" {
  source = "./modules/vpc"

  vpc_cidr           = "10.0.0.0/16"
  environment        = "production"
  public_subnets     = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets    = ["10.0.10.0/24", "10.0.11.0/24"]
  availability_zones = ["us-east-1a", "us-east-1b"]
}

module "database" {
  source = "./modules/rds"

  identifier     = "production-db"
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.r6g.large"
  subnet_ids     = module.networking.private_subnet_ids
  vpc_id         = module.networking.vpc_id
}

module "application" {
  source = "./modules/ecs-service"

  cluster_name  = "production"
  service_name  = "web-api"
  image         = "my-org/web-api:v1.2.3"
  subnet_ids    = module.networking.private_subnet_ids
  vpc_id        = module.networking.vpc_id
  db_endpoint   = module.database.endpoint
}

module "monitoring" {
  source = "./modules/cloudwatch"

  environment    = "production"
  ecs_cluster    = module.application.cluster_arn
  rds_identifier = module.database.identifier
  alarm_sns_topic = var.alarm_sns_topic_arn
}
```

---

## State Management

### What is Terraform State

Terraform state is a JSON file that maps your configuration to real-world infrastructure. It tracks resource metadata, dependency information, and attribute values needed to plan and apply changes.

```
  Terraform State — Mapping Configuration to Reality
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Configuration (.tf files)          State (terraform.tfstate)
  ┌─────────────────────────┐        ┌───────────────────────────┐
  │ resource "aws_vpc" "m"  │        │ aws_vpc.main:             │
  │   cidr = "10.0.0.0/16"  │◀──────▶│   id   = "vpc-abc123"    │
  │                          │        │   cidr = "10.0.0.0/16"   │
  │ resource "aws_subnet" "s"│        │                           │
  │   cidr = "10.0.1.0/24"  │◀──────▶│ aws_subnet.sub:           │
  │   vpc  = aws_vpc.m.id   │        │   id   = "subnet-def456" │
  └─────────────────────────┘        │   vpc  = "vpc-abc123"    │
                                     └───────────────────────────┘
                                                │
                                                │ maps to
                                                ▼
                                     ┌───────────────────────────┐
                                     │    Real Infrastructure    │
                                     │                           │
                                     │  VPC: vpc-abc123          │
                                     │  Subnet: subnet-def456   │
                                     └───────────────────────────┘
```

**Why state matters:**

- **Performance** — Terraform queries state instead of making API calls for every resource during planning
- **Dependency tracking** — State records the dependency graph for correct ordering of operations
- **Resource mapping** — State maps resource addresses (e.g., `aws_vpc.main`) to real resource IDs
- **Drift detection** — Terraform compares state to real infrastructure during `terraform plan`

### Local State

By default, Terraform stores state in a local file called `terraform.tfstate`:

```bash
# After terraform apply, state is written to the current directory
$ ls -la terraform.tfstate
-rw-r--r-- 1 user user 12345 Jan 15 10:30 terraform.tfstate

# The state file is JSON
$ head -20 terraform.tfstate
{
  "version": 4,
  "terraform_version": "1.5.0",
  "serial": 3,
  "lineage": "abc123-def456-...",
  "outputs": {},
  "resources": [...]
}
```

> **Best Practice:** Never use local state for team projects. Local state has no locking, no shared access, and is easily lost or corrupted. Always use a remote backend for any infrastructure managed by more than one person.

### Remote Backends

Remote backends store state in a shared, lockable location:

```hcl
# AWS S3 + DynamoDB backend
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state"
    key            = "production/networking/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}

# Azure Blob Storage backend
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate"
    container_name       = "tfstate"
    key                  = "production/networking/terraform.tfstate"
  }
}

# Google Cloud Storage backend
terraform {
  backend "gcs" {
    bucket = "my-company-terraform-state"
    prefix = "production/networking"
  }
}

# Terraform Cloud backend
terraform {
  cloud {
    organization = "my-company"

    workspaces {
      name = "production-networking"
    }
  }
}
```

### State Backend Comparison

| Backend | Locking | Encryption | Versioning | Cost | Best For |
|---|---|---|---|---|---|
| Local file | None | None (plaintext) | None | Free | Solo dev, learning |
| S3 + DynamoDB | DynamoDB | AES-256 (SSE) | S3 versioning | Low | AWS teams |
| Azure Blob | Blob lease | AES-256 | Blob versioning | Low | Azure teams |
| GCS | Built-in | AES-256 (default) | Object versioning | Low | GCP teams |
| Terraform Cloud | Built-in | At rest + transit | Built-in | Free tier / Paid | Any team, enterprise |
| PostgreSQL | Advisory locks | TLS in transit | None built-in | DB hosting cost | Self-hosted teams |

> **Best Practice:** Always enable state encryption, versioning, and locking. Use a dedicated storage account/bucket for state files with restricted IAM permissions. Enable versioning so you can recover from state corruption.

### Workspaces

Workspaces allow multiple state files within a single configuration. Each workspace has its own independent state:

```bash
# List workspaces
$ terraform workspace list
* default

# Create and switch to a new workspace
$ terraform workspace new staging
Created and switched to workspace "staging"!

# Switch between workspaces
$ terraform workspace select production

# Show current workspace
$ terraform workspace show
production
```

```hcl
# Using workspace name in configuration
locals {
  environment = terraform.workspace

  instance_type = {
    default     = "t3.micro"
    staging     = "t3.small"
    production  = "t3.large"
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = lookup(local.instance_type, terraform.workspace, "t3.micro")

  tags = {
    Name        = "web-${terraform.workspace}"
    Environment = terraform.workspace
  }
}
```

> **Best Practice:** Many teams prefer separate directories or separate Terraform Cloud workspaces over CLI workspaces. CLI workspaces share the same configuration and backend, which can lead to accidental changes to the wrong environment. Use CLI workspaces for simple use cases; use directory-based separation for production.

---

## Terraform CLI

### Core Workflow Commands

```bash
# terraform init — Initialize a working directory
# Downloads providers, initializes backend, installs modules
$ terraform init

# terraform init with backend configuration
$ terraform init \
    -backend-config="bucket=my-state-bucket" \
    -backend-config="key=prod/terraform.tfstate" \
    -backend-config="region=us-east-1"

# terraform init — upgrade providers to latest allowed versions
$ terraform init -upgrade

# terraform plan — Preview changes
# Compares configuration to state and shows what will change
$ terraform plan

# Save a plan to a file for later apply
$ terraform plan -out=tfplan

# Plan for a specific target
$ terraform plan -target=aws_instance.web

# Plan with variable overrides
$ terraform plan -var="environment=production" -var="instance_count=5"

# terraform apply — Apply changes
# Creates, updates, or destroys infrastructure to match configuration
$ terraform apply

# Apply a saved plan (no confirmation prompt)
$ terraform apply tfplan

# Apply with auto-approve (use in CI/CD only)
$ terraform apply -auto-approve

# terraform destroy — Destroy all managed infrastructure
$ terraform destroy

# Destroy specific resources
$ terraform destroy -target=aws_instance.web

# Destroy with auto-approve (use in CI/CD only)
$ terraform destroy -auto-approve
```

### Formatting and Validation

```bash
# terraform fmt — Format HCL files to canonical style
$ terraform fmt

# Format recursively
$ terraform fmt -recursive

# Check formatting without making changes (useful in CI)
$ terraform fmt -check -recursive

# terraform validate — Validate configuration syntax and internal consistency
$ terraform validate
Success! The configuration is valid.
```

### State Commands

```bash
# terraform state list — List all resources in state
$ terraform state list
aws_vpc.main
aws_subnet.public[0]
aws_subnet.public[1]
aws_instance.web

# terraform state show — Show attributes of a single resource
$ terraform state show aws_vpc.main

# terraform state mv — Rename or move a resource in state
# Useful when refactoring (e.g., renaming a resource or moving into a module)
$ terraform state mv aws_instance.web aws_instance.api

# Move a resource into a module
$ terraform state mv aws_vpc.main module.networking.aws_vpc.this

# terraform state rm — Remove a resource from state (without destroying it)
# The resource continues to exist but Terraform no longer manages it
$ terraform state rm aws_instance.legacy

# terraform state pull — Download and display the remote state
$ terraform state pull > state_backup.json

# terraform state push — Upload a local state file to the remote backend
$ terraform state push state_backup.json
```

> **Best Practice:** Always back up state before running `state mv` or `state rm` commands. Use `terraform state pull > backup.json` to create a backup. These operations modify state directly and cannot be undone through normal Terraform operations.

### Import Command

```bash
# terraform import — Import an existing resource into Terraform state
# Syntax: terraform import <resource_address> <resource_id>

# Import an existing AWS VPC
$ terraform import aws_vpc.main vpc-abc123

# Import an existing Azure Resource Group
$ terraform import azurerm_resource_group.example \
    /subscriptions/sub-id/resourceGroups/rg-example

# Import an existing GCP bucket
$ terraform import google_storage_bucket.assets my-company-assets-bucket
```

---

## Terraform Cloud and Enterprise

### Remote Execution

Terraform Cloud provides a managed platform for running Terraform operations. Instead of running `terraform plan` and `terraform apply` on local machines, operations run in Terraform Cloud's managed environment:

```
  Terraform Cloud Workflow
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Developer                    Terraform Cloud
  ┌────────────┐               ┌─────────────────────────────────┐
  │            │  git push     │                                 │
  │  Write HCL │──────────────▶│  1. Detect VCS change           │
  │  files     │               │  2. Queue plan                  │
  │            │               │  3. Run terraform plan           │
  └────────────┘               │  4. Show plan in UI/PR          │
                               │  5. Wait for approval            │
  ┌────────────┐               │  6. Run terraform apply           │
  │            │  Review plan  │  7. Update state                 │
  │  Approve   │◀─────────────▶│  8. Notify on completion        │
  │  in UI     │               │                                 │
  └────────────┘               └─────────────────────────────────┘
```

```hcl
# Configure Terraform Cloud as the backend
terraform {
  cloud {
    organization = "my-company"

    workspaces {
      name = "production-infrastructure"
    }
  }
}
```

Key features of remote execution:

| Feature | Description |
|---|---|
| Consistent environment | Every run uses the same Terraform version and OS |
| State management | Built-in remote state with locking and encryption |
| Run history | Full audit log of every plan and apply |
| Cost estimation | Estimate cloud costs before applying changes |
| Notifications | Slack, email, or webhook notifications for run status |

### VCS Integration

Terraform Cloud connects directly to version control systems (GitHub, GitLab, Bitbucket, Azure DevOps) to trigger runs automatically:

```
  VCS-Driven Workflow
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Pull Request Opened              Merged to Main
  ┌──────────────────┐             ┌──────────────────┐
  │  Speculative Plan │             │  Confirmed Apply  │
  │  (plan only,     │             │  (plan + apply)   │
  │   no apply)      │             │                   │
  │                  │             │                   │
  │  ✓ Shows diff    │             │  ✓ Applies        │
  │    in PR comment │             │    changes        │
  │  ✓ No state      │             │  ✓ Updates state  │
  │    changes       │             │  ✓ Sends alerts   │
  └──────────────────┘             └──────────────────┘
```

### Policy as Code with Sentinel

Sentinel is HashiCorp's policy-as-code framework for enforcing governance rules on Terraform runs. Policies run between the plan and apply phases:

```python
# Example Sentinel policy: require tags on all AWS instances
import "tfplan/v2" as tfplan

required_tags = ["Environment", "Team", "ManagedBy"]

ec2_instances = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_instance" and
  rc.mode is "managed" and
  (rc.change.actions contains "create" or rc.change.actions contains "update")
}

tags_contain_required = rule {
  all ec2_instances as _, instance {
    all required_tags as tag {
      instance.change.after.tags contains tag
    }
  }
}

main = rule {
  tags_contain_required
}
```

```python
# Example Sentinel policy: restrict instance types
import "tfplan/v2" as tfplan

allowed_types = ["t3.micro", "t3.small", "t3.medium", "t3.large"]

ec2_instances = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_instance" and
  rc.mode is "managed" and
  (rc.change.actions contains "create" or rc.change.actions contains "update")
}

instance_type_allowed = rule {
  all ec2_instances as _, instance {
    instance.change.after.instance_type in allowed_types
  }
}

main = rule {
  instance_type_allowed
}
```

| Enforcement Level | Behavior |
|---|---|
| Advisory | Warn but allow apply to proceed |
| Soft Mandatory | Require override approval to proceed |
| Hard Mandatory | Block apply — no override possible |

---

## Data Sources and Dynamic Blocks

### Data Source Patterns

Data sources enable reading information from external systems and existing infrastructure:

```hcl
# Pattern 1: Look up existing resources to reference them
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

resource "aws_subnet" "new" {
  vpc_id     = data.aws_vpc.existing.id
  cidr_block = "10.0.100.0/24"
}

# Pattern 2: Query available options
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "multi_az" {
  count = length(data.aws_availability_zones.available.names)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}

# Pattern 3: Read remote state from another Terraform workspace
data "terraform_remote_state" "networking" {
  backend = "s3"

  config = {
    bucket = "my-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "web" {
  subnet_id = data.terraform_remote_state.networking.outputs.public_subnet_ids[0]
}

# Pattern 4: Generate IAM policy documents
data "aws_iam_policy_document" "s3_read" {
  statement {
    effect = "Allow"
    actions = [
      "s3:GetObject",
      "s3:ListBucket",
    ]
    resources = [
      aws_s3_bucket.data.arn,
      "${aws_s3_bucket.data.arn}/*",
    ]
  }
}

resource "aws_iam_policy" "s3_read" {
  name   = "s3-read-access"
  policy = data.aws_iam_policy_document.s3_read.json
}
```

### Dynamic Blocks

Dynamic blocks generate repeated nested blocks within a resource. They replace copy-pasted nested blocks with a loop:

```hcl
# Without dynamic blocks — repetitive and hard to maintain
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }
}

# With dynamic blocks — DRY and data-driven
variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))
  default = [
    {
      port        = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTP"
    },
    {
      port        = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTPS"
    },
    {
      port        = 22
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/8"]
      description = "SSH from internal"
    },
  ]
}

resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.ingress_rules

    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound traffic"
  }
}
```

> **Best Practice:** Use dynamic blocks sparingly. They reduce repetition but can make configurations harder to read and debug. Prefer explicit nested blocks when there are only 2-3 entries. Reserve dynamic blocks for data-driven configurations with many entries.

---

## Provisioners

### When to Use Provisioners

Provisioners execute scripts or commands on local or remote machines as part of resource creation or destruction. Terraform provides three types:

```hcl
# local-exec — runs a command on the machine running Terraform
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  provisioner "local-exec" {
    command = "echo ${self.private_ip} >> private_ips.txt"
  }
}

# remote-exec — runs commands on the remote resource via SSH or WinRM
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
    ]
  }
}

# file — copies files or directories to a remote machine
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }

  provisioner "file" {
    source      = "configs/nginx.conf"
    destination = "/tmp/nginx.conf"
  }
}
```

### Why to Avoid Provisioners

HashiCorp explicitly recommends **avoiding provisioners** whenever possible. They introduce several problems:

| Problem | Description |
|---|---|
| Not declarative | Provisioners run arbitrary commands — Terraform cannot model or plan their effects |
| Not idempotent | Running the same provisioner twice may produce different results or errors |
| Not in state | Provisioner results are not tracked in state — drift cannot be detected |
| Fragile | Network issues, SSH timeouts, and script errors cause failures that are hard to recover from |
| Taint-only recovery | If a provisioner fails, the resource is tainted and must be destroyed and recreated |

### Alternatives to Provisioners

| Instead of... | Use... |
|---|---|
| `remote-exec` to configure a VM | Cloud-init (`user_data`), Packer images, Ansible |
| `local-exec` to trigger builds | CI/CD pipelines, `null_resource` + `triggers` |
| `file` provisioner to upload config | Cloud-init, S3/Blob download in user_data |
| `remote-exec` to install packages | Pre-baked AMIs with Packer, container images |

```hcl
# Preferred: Use cloud-init / user_data instead of provisioners
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  user_data = <<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl enable nginx
    systemctl start nginx
  EOF

  user_data_replace_on_change = true
}

# Preferred: Use templatefile for complex user_data
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  user_data = templatefile("${path.module}/templates/user_data.sh.tpl", {
    environment = var.environment
    app_version = var.app_version
    db_endpoint = module.database.endpoint
  })
}
```

---

## Terraform Import and Migration

### Importing Existing Resources

Terraform import brings existing infrastructure under Terraform management. This is essential when adopting Terraform for infrastructure that was created manually or by another tool.

```bash
# Step 1: Write the resource configuration in your .tf file
# (You must write the configuration BEFORE importing)

# Step 2: Import the resource
$ terraform import aws_instance.web i-1234567890abcdef0

# Step 3: Run terraform plan to verify the imported state matches your config
$ terraform plan
# Address any differences between your configuration and the actual resource

# Step 4: Iterate until terraform plan shows no changes
```

### Import Blocks (Terraform 1.5+)

Terraform 1.5 introduced declarative import blocks that integrate with the plan/apply workflow:

```hcl
# Import an existing VPC
import {
  to = aws_vpc.main
  id = "vpc-abc123"
}

# Import an existing S3 bucket
import {
  to = aws_s3_bucket.logs
  id = "my-company-logs-bucket"
}

# Import an existing Azure Resource Group
import {
  to = azurerm_resource_group.legacy
  id = "/subscriptions/sub-id/resourceGroups/rg-legacy"
}

# Generate configuration for imported resources (Terraform 1.5+)
# terraform plan -generate-config-out=generated_resources.tf
```

The `import` block workflow:

```bash
# Step 1: Add import blocks to your configuration
# Step 2: Run plan with config generation
$ terraform plan -generate-config-out=generated_resources.tf

# Step 3: Review and refine the generated configuration
# Step 4: Remove the -generate-config-out flag and run plan again
$ terraform plan

# Step 5: Apply when the plan shows no unexpected changes
$ terraform apply
```

> **Best Practice:** Import blocks are preferred over `terraform import` CLI commands because they are declarative, reviewable in PRs, and integrate with the normal plan/apply workflow. They also support the `-generate-config-out` flag to scaffold configuration for imported resources.

### Migration Strategies

When migrating existing infrastructure to Terraform:

| Strategy | Description | Risk | Best For |
|---|---|---|---|
| Import in place | Import resources one by one into Terraform state | Low | Small environments, gradual adoption |
| Parallel management | Run Terraform alongside existing tools during transition | Medium | Large environments with active changes |
| Recreate | Let Terraform create new resources, migrate data, decommission old | Higher | When refactoring architecture |
| State surgery | Manipulate state files to restructure resource addresses | Medium | Refactoring modules or renaming resources |

```bash
# Common migration workflow
# 1. Start with read-only data sources to reference existing resources
# 2. Gradually import resources into Terraform management
# 3. Use moved blocks to refactor without destroying resources

# moved block — refactor resource addresses without destroy/recreate
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}

moved {
  from = aws_vpc.main
  to   = module.networking.aws_vpc.this
}
```

---

## Next Steps

Continue your Infrastructure as Code learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | IaC Fundamentals | Core IaC concepts, declarative vs imperative, tool landscape |
| [02-PULUMI.md](02-PULUMI.md) | Pulumi | General-purpose languages for IaC, stacks, automation API |
| [03-CLOUDFORMATION.md](03-CLOUDFORMATION.md) | AWS CloudFormation | AWS-native IaC, nested stacks, drift detection |
| [04-STATE-MANAGEMENT.md](04-STATE-MANAGEMENT.md) | State Management | Remote state, locking, import, state surgery |
| [05-MODULES-AND-REUSE.md](05-MODULES-AND-REUSE.md) | Modules and Reuse | Module design, versioning, registries |
| [06-TESTING.md](06-TESTING.md) | IaC Testing | Policy as code, plan validation, integration testing |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Directory structure, naming, CI/CD integration |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial Terraform documentation |
