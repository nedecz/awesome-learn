# State Management

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [What is State](#what-is-state)
   - [Definition and Purpose](#definition-and-purpose)
   - [How IaC Tools Track Real-World Resources](#how-iac-tools-track-real-world-resources)
   - [State as Source of Truth](#state-as-source-of-truth)
   - [State Across IaC Tools](#state-across-iac-tools)
3. [Local vs Remote State](#local-vs-remote-state)
   - [Local State Files](#local-state-files)
   - [Why Remote State is Essential](#why-remote-state-is-essential)
   - [Remote Backend Options](#remote-backend-options)
   - [Backend Comparison](#backend-comparison)
   - [Configuring Remote Backends](#configuring-remote-backends)
4. [State Locking](#state-locking)
   - [Why Locking Matters](#why-locking-matters)
   - [How State Locking Works](#how-state-locking-works)
   - [DynamoDB Locking for Terraform](#dynamodb-locking-for-terraform)
   - [Lock Timeouts and Force-Unlock](#lock-timeouts-and-force-unlock)
5. [State File Structure](#state-file-structure)
   - [Terraform State JSON Anatomy](#terraform-state-json-anatomy)
   - [Resource Instance Tracking](#resource-instance-tracking)
   - [Sensitive Data in State](#sensitive-data-in-state)
   - [Encryption at Rest](#encryption-at-rest)
6. [Workspaces](#workspaces)
   - [Terraform Workspaces](#terraform-workspaces)
   - [Pulumi Stacks](#pulumi-stacks)
   - [Environment Isolation Strategies](#environment-isolation-strategies)
   - [Workspace Anti-Patterns](#workspace-anti-patterns)
7. [State Import](#state-import)
   - [Bringing Existing Resources Under Management](#bringing-existing-resources-under-management)
   - [Terraform Import](#terraform-import)
   - [Import Blocks (Terraform 1.5+)](#import-blocks-terraform-15)
   - [Pulumi Import](#pulumi-import)
   - [CloudFormation Resource Import](#cloudformation-resource-import)
8. [State Surgery](#state-surgery)
   - [When State Surgery is Needed](#when-state-surgery-is-needed)
   - [terraform state mv](#terraform-state-mv)
   - [terraform state rm](#terraform-state-rm)
   - [terraform state pull and push](#terraform-state-pull-and-push)
   - [Recovering Corrupted State](#recovering-corrupted-state)
9. [State Migration](#state-migration)
   - [Moving Between Backends](#moving-between-backends)
   - [Splitting Monolithic State](#splitting-monolithic-state)
   - [Merging States](#merging-states)
   - [Migration Checklist](#migration-checklist)
10. [Drift Detection and Reconciliation](#drift-detection-and-reconciliation)
    - [What is Drift](#what-is-drift)
    - [Terraform Plan Drift Detection](#terraform-plan-drift-detection)
    - [CloudFormation Drift Detection](#cloudformation-drift-detection)
    - [Remediation Strategies](#remediation-strategies)
11. [State Security](#state-security)
    - [Encrypting State](#encrypting-state)
    - [Access Control](#access-control)
    - [Audit Logging](#audit-logging)
    - [Sensitive Values in State](#sensitive-values-in-state)
12. [State at Scale](#state-at-scale)
    - [Large State Files](#large-state-files)
    - [Performance Implications](#performance-implications)
    - [State Splitting Strategies](#state-splitting-strategies)
13. [Multi-Account and Multi-Region State](#multi-account-and-multi-region-state)
    - [Patterns for Multi-Account State](#patterns-for-multi-account-state)
    - [Multi-Region State Architecture](#multi-region-state-architecture)
    - [Cross-Account State Access](#cross-account-state-access)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **State Management** in Infrastructure as Code — the mechanisms by which IaC tools track, store, and synchronize the relationship between your declared configurations and the real-world resources they manage. It covers state fundamentals, remote backends, locking, workspaces, import and migration strategies, drift detection, security, and patterns for managing state at scale across multiple accounts and regions.

### Target Audience

- **DevOps Engineers** managing Terraform, Pulumi, or CloudFormation state across teams and environments
- **Platform Engineers** designing state backend architectures, workspace strategies, and self-service infrastructure platforms
- **Site Reliability Engineers (SREs)** troubleshooting state corruption, drift, and operational issues in production infrastructure
- **Developers** working with IaC tools who need to understand how state affects their workflows and collaboration
- **Architects** evaluating state management strategies for multi-account, multi-region, and enterprise-scale deployments

### Scope

- What state is, why it exists, and how IaC tools use it to manage infrastructure
- Local vs remote state storage: backends, configuration, and trade-offs
- State locking mechanisms to prevent concurrent modification
- State file structure, sensitive data handling, and encryption
- Workspace and stack strategies for environment isolation
- Importing existing resources into state and performing state surgery
- Migrating state between backends, splitting, and merging
- Drift detection and reconciliation across tools
- State security: encryption, access control, and audit logging
- Performance and organizational patterns for state at scale

---

## What is State

### Definition and Purpose

**State** is the persistent record that an IaC tool maintains to map your declared configuration to the real-world resources it has created or is managing. Without state, the tool has no way to know which cloud resources correspond to which configuration blocks, what their current attributes are, or what changes need to be applied.

State serves several critical purposes:

| Purpose | Description |
|---|---|
| **Resource Mapping** | Associates configuration blocks with real infrastructure resource IDs |
| **Attribute Storage** | Records current attributes of managed resources (IPs, ARNs, IDs) |
| **Dependency Tracking** | Maintains the dependency graph between resources |
| **Change Detection** | Enables comparison between desired state and actual state |
| **Performance** | Caches resource attributes to avoid querying every resource on every run |

Consider a simple Terraform configuration that creates an S3 bucket:

```hcl
resource "aws_s3_bucket" "data" {
  bucket = "my-app-data-bucket"

  tags = {
    Environment = "production"
    Team        = "platform"
  }
}
```

After `terraform apply`, the state records:

- The S3 bucket resource type and name (`aws_s3_bucket.data`)
- The real AWS bucket name (`my-app-data-bucket`)
- The bucket ARN (`arn:aws:s3:::my-app-data-bucket`)
- The region, tags, and all other attributes returned by the AWS API

### How IaC Tools Track Real-World Resources

Each IaC tool approaches state differently, but they all solve the same fundamental problem: **bridging the gap between declared configuration and real infrastructure**.

```
┌─────────────────────────────────────────────────────────┐
│  Configuration ──▶ State ──▶ Cloud APIs                  │
│       │              │            │                       │
│       └──── Plan/Diff (compare) ◀─┘                      │
│                  │                                        │
│               Apply (execute)                             │
└─────────────────────────────────────────────────────────┘
```

The workflow:

1. **Read** the current state to understand what resources exist
2. **Read** the configuration to understand the desired state
3. **Query** cloud APIs to verify actual state (refresh)
4. **Compare** desired state against current state to produce a diff
5. **Apply** the changes and update the state with new resource attributes

### State as Source of Truth

State acts as the **source of truth** for your IaC tool — not your configuration files, and not the cloud provider's actual state. This distinction is critical:

- When **state and reality diverge** (someone manually deletes a resource), the IaC tool detects this **drift** during a refresh and plans corrective action
- When **configuration and state diverge** (you change a configuration value), the tool plans an update
- When **all three agree**, no changes are planned

### State Across IaC Tools

| Tool | State Mechanism | Storage | Format |
|---|---|---|---|
| **Terraform** | Explicit state file (`terraform.tfstate`) | Local file or remote backend | JSON |
| **Pulumi** | Managed state via service or self-hosted | Pulumi Cloud, S3, local, etc. | JSON (encrypted) |
| **CloudFormation** | AWS-managed — no user-accessible state file | AWS internal storage | N/A (managed) |
| **Crossplane** | Kubernetes custom resources | etcd (via Kubernetes) | YAML/JSON |
| **Ansible** | Stateless — queries current state each run | N/A | N/A |

---

## Local vs Remote State

### Local State Files

By default, Terraform stores state in a local file named `terraform.tfstate` in your working directory:

```
my-project/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfstate          # Current state
└── terraform.tfstate.backup   # Previous state (automatic backup)
```

Local state is simple and requires no additional infrastructure, but it has severe limitations:

| Limitation | Impact |
|---|---|
| **No collaboration** | Only one person can work on infrastructure at a time |
| **No locking** | Concurrent runs can corrupt the state file |
| **No encryption** | State file on disk contains secrets in plaintext |
| **No versioning** | Accidental deletion or corruption is difficult to recover from |
| **No audit trail** | No record of who changed what and when |

Local state is acceptable only for:

- Learning and experimentation
- Single-developer personal projects
- Disposable or ephemeral environments

### Why Remote State is Essential

For any team or production use, remote state is **mandatory**. Remote backends provide:

- **Shared access** — team members read and write to the same state
- **State locking** — prevents concurrent modifications
- **Encryption** — state is encrypted in transit and at rest
- **Versioning** — state history enables rollback and audit
- **Durability** — cloud storage with redundancy protects against data loss

### Remote Backend Options

#### AWS: S3 + DynamoDB

The most widely used Terraform backend in AWS environments:

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "production/networking/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
    kms_key_id     = "alias/terraform-state"
  }
}
```

#### Azure: Azure Blob Storage

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate"
    container_name       = "tfstate"
    key                  = "production/networking/terraform.tfstate"
    use_oidc             = true
  }
}
```

#### GCP: Google Cloud Storage

```hcl
terraform {
  backend "gcs" {
    bucket = "mycompany-terraform-state"
    prefix = "production/networking"
  }
}
```

#### Terraform Cloud / HCP Terraform

```hcl
terraform {
  cloud {
    organization = "mycompany"

    workspaces {
      name = "production-networking"
    }
  }
}
```

#### Pulumi Cloud

Pulumi stores state in its managed service by default:

```bash
# Pulumi uses its cloud service by default
pulumi login

# Or use a self-managed backend
pulumi login s3://my-pulumi-state-bucket
pulumi login azblob://my-pulumi-container
pulumi login gs://my-pulumi-state-bucket
pulumi login file://~/.pulumi-local
```

### Backend Comparison

| Backend | Locking | Encryption | Versioning | Cost | Setup Complexity |
|---|---|---|---|---|---|
| **Local file** | None | None | Manual backups | Free | None |
| **S3 + DynamoDB** | DynamoDB | SSE-S3/KMS | S3 versioning | Low (~$1/mo) | Medium |
| **Azure Blob** | Blob leases | Azure SSE | Blob versioning | Low (~$1/mo) | Medium |
| **GCS** | Built-in | Google-managed | Object versioning | Low (~$1/mo) | Medium |
| **Terraform Cloud** | Built-in | Built-in | Built-in | Free tier available | Low |
| **Pulumi Cloud** | Built-in | Built-in (secrets) | Built-in | Free tier available | Low |
| **Consul** | Built-in | TLS | None (use snapshots) | Self-hosted | High |
| **pg (PostgreSQL)** | Advisory locks | TLS | None (use backups) | Self-hosted | High |

### Configuring Remote Backends

Setting up an S3 backend with all recommended features — a common production pattern:

```hcl
# bootstrap/main.tf — Create the state backend infrastructure first
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state"
  lifecycle { prevent_destroy = true }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration { status = "Enabled" }
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
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute { name = "LockID"; type = "S" }
}

resource "aws_kms_key" "terraform_state" {
  description             = "KMS key for Terraform state encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}
```

---

## State Locking

### Why Locking Matters

Without state locking, two team members (or two CI/CD pipelines) can run `terraform apply` at the same time, leading to:

- **Race conditions** — both reads see the same state, both try to create the same resource
- **State corruption** — concurrent writes produce an invalid state file
- **Resource duplication** — two applies may each create their own copy of a resource
- **Data loss** — one apply overwrites the other's state changes

```
Without Locking:

  Developer A                    Developer B
       │                              │
       ├─ terraform plan              │
       │  (reads state v1)            │
       │                              ├─ terraform plan
       │                              │  (reads state v1)
       ├─ terraform apply             │
       │  (creates resource X)        │
       │  (writes state v2)           │
       │                              ├─ terraform apply
       │                              │  (creates resource X again!)
       │                              │  (writes state v2' — CONFLICT)
       │                              │
       ▼                              ▼
   State is now corrupted or inconsistent
```

### How State Locking Works

State locking follows a simple protocol:

1. Before any state-modifying operation, the tool attempts to **acquire a lock**
2. If the lock is already held, the operation **waits or fails** with a clear message
3. After the operation completes (success or failure), the lock is **released**
4. If the process crashes, the lock remains — requiring manual intervention

The lock record contains a unique lock ID, who holds it, when it was acquired, and the operation type.

### DynamoDB Locking for Terraform

When using S3 + DynamoDB, Terraform uses DynamoDB's conditional writes for locking:

```hcl
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute { name = "LockID"; type = "S" }
}
```

The lock entry stored in DynamoDB contains the lock ID, operation type, who holds it, the Terraform version, timestamp, and state file path.

### Lock Timeouts and Force-Unlock

When a lock cannot be acquired, Terraform retries with a configurable timeout:

```bash
# Default lock timeout is 0s (fail immediately)
terraform plan -lock-timeout=5m

# Apply with a 10-minute lock timeout
terraform apply -lock-timeout=10m

# Skip locking entirely (DANGEROUS — use only in emergencies)
terraform apply -lock=false
```

If a process crashes and leaves a stale lock, use `force-unlock`:

```bash
# Get the lock ID from the error message
# Error: Error locking state: Error acquiring the state lock
# Lock Info:
#   ID:        a1b2c3d4-e5f6-7890-abcd-ef1234567890

# Force-unlock requires the lock ID
terraform force-unlock a1b2c3d4-e5f6-7890-abcd-ef1234567890

# Skip confirmation prompt
terraform force-unlock -force a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

> **Warning**: Only use `force-unlock` when you are absolutely certain no other operation is running. Unlocking an active operation can cause state corruption.

---

## State File Structure

### Terraform State JSON Anatomy

The Terraform state file is a JSON document with a well-defined structure:

```json
{
  "version": 4,
  "terraform_version": "1.7.0",
  "serial": 42,
  "lineage": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "outputs": {
    "vpc_id": {
      "value": "vpc-0123456789abcdef0",
      "type": "string"
    },
    "subnet_ids": {
      "value": ["subnet-aaa", "subnet-bbb", "subnet-ccc"],
      "type": ["list", "string"]
    }
  },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_vpc",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "vpc-0123456789abcdef0",
            "arn": "arn:aws:ec2:us-east-1:123456789012:vpc/vpc-0123456789abcdef0",
            "cidr_block": "10.0.0.0/16",
            "tags": {
              "Name": "main-vpc",
              "Environment": "production"
            }
          }
        }
      ]
    }
  ]
}
```

Key fields explained:

| Field | Description |
|---|---|
| `version` | State file format version (currently 4) |
| `terraform_version` | Terraform version that last wrote this state |
| `serial` | Incremented on every state write — used for conflict detection |
| `lineage` | Unique ID for this state's history — prevents accidentally overwriting unrelated state |
| `outputs` | Values exported by `output` blocks |
| `resources` | Array of all managed and data source resources |

### Resource Instance Tracking

Each resource in state tracks its full attributes, provider, schema version, and — when using `count` or `for_each` — an `index_key` to distinguish instances (numeric for `count`, string key for `for_each`).

### Sensitive Data in State

**State files contain sensitive data in plaintext.** Any attribute marked as `sensitive` by the provider, plus all values from `sensitive` variables, appear in the state file unmasked. For example, an `aws_db_instance` resource stores the database password in plain text in state.

This means:

- **Never commit state files to version control** — add `*.tfstate` to `.gitignore`
- **Always use encrypted remote backends** for production state
- **Restrict access** to state storage to only those who need it
- **Use external secret managers** (Vault, AWS Secrets Manager) instead of passing secrets through Terraform variables when possible

### Encryption at Rest

Each backend provides its own encryption mechanism:

| Backend | Encryption Method | Configuration |
|---|---|---|
| **S3** | SSE-S3, SSE-KMS, or SSE-C | `encrypt = true`, `kms_key_id` |
| **Azure Blob** | Azure Storage Service Encryption (256-bit AES) | Enabled by default |
| **GCS** | Google-managed or CMEK | Default or `encryption_key` |
| **Terraform Cloud** | HashiCorp Vault Transit encryption | Automatic |
| **Pulumi Cloud** | Per-value encryption with passphrase or provider | Automatic |

---

## Workspaces

### Terraform Workspaces

Terraform workspaces allow you to manage multiple distinct state files from a single configuration directory. Each workspace has its own state:

```bash
# List workspaces
terraform workspace list
# * default
#   staging
#   production

# Create a new workspace
terraform workspace new staging

# Switch to an existing workspace
terraform workspace select production

# Show current workspace
terraform workspace show
# production

# Delete a workspace (must switch away first)
terraform workspace select default
terraform workspace delete staging
```

Use `terraform.workspace` in your configuration to customize behavior per workspace:

```hcl
locals {
  environment   = terraform.workspace
  instance_type = {
    default    = "t3.micro"
    staging    = "t3.small"
    production = "t3.large"
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.instance_type[local.environment]
  tags = { Name = "web-${local.environment}" }
}
```

State file paths in the remote backend with workspaces follow the convention `env:/<workspace>/<key>`, with the `default` workspace using the key directly.

### Pulumi Stacks

Pulumi uses **stacks** as its equivalent of workspaces — each stack is an independently configurable instance of a Pulumi program:

```bash
# Create and manage stacks
pulumi stack init dev
pulumi stack init staging
pulumi stack init production

# List stacks
pulumi stack ls

# Select a stack and set configuration
pulumi stack select production
pulumi config set aws:region us-east-1
pulumi config set instanceType t3.large
pulumi config set --secret dbPassword SuperSecret123
```

Pulumi stacks provide stronger isolation than Terraform workspaces because each stack has its own configuration file (`Pulumi.<stack>.yaml`) and secrets provider.

### Environment Isolation Strategies

There are two primary approaches to environment isolation:

```
Strategy 1: Workspaces/Stacks        Strategy 2: Directory Per Environment
(Same code, different state)          (Different code, different state)

 ┌───────────────────────────┐        ┌────────┐ ┌────────┐ ┌──────────┐
 │   Single Configuration    │        │  dev/  │ │staging/│ │production│
 │                           │        │main.tf │ │main.tf │ │ main.tf  │
 │ ┌─────┐ ┌─────┐ ┌──────┐ │        │vars.tf │ │vars.tf │ │ vars.tf  │
 │ │ dev │ │stag │ │ prod │ │        │ state  │ │ state  │ │  state   │
 │ └─────┘ └─────┘ └──────┘ │        └────────┘ └────────┘ └──────────┘
 └───────────────────────────┘
```

| Strategy | Pros | Cons |
|---|---|---|
| **Workspaces / Stacks** | DRY — single source of truth; easy to keep environments in sync | Less flexibility for per-environment differences; risk of applying to wrong workspace |
| **Directory per environment** | Full control per environment; clear separation; independent state keys | Code duplication; drift between environments; more files to maintain |
| **Modules + composition** | DRY modules with per-environment root configs; best of both | More initial setup; requires module versioning discipline |

### Workspace Anti-Patterns

Workspaces work well for environments that share the same configuration structure but differ in scale or settings. They are **not** a good fit for:

- **Completely different infrastructure** (e.g., networking vs application) — use separate root modules instead
- **Different cloud accounts with different access patterns** — use separate backend configurations
- **Teams that need independent deployment cadences** — separate root modules give independent state and CI/CD

---

## State Import

### Bringing Existing Resources Under Management

Organizations rarely start with a clean slate. Existing infrastructure needs to be brought under IaC management without recreating resources. The workflow is: identify the resource and its cloud ID, write the IaC configuration, run the import command, then adjust configuration until `plan` shows no changes.

### Terraform Import

The traditional Terraform import workflow uses the CLI:

```bash
# Import an existing VPC into Terraform state
terraform import aws_vpc.main vpc-0123456789abcdef0

# Import an S3 bucket
terraform import aws_s3_bucket.data my-app-data-bucket

# Import a resource within a module
terraform import module.networking.aws_subnet.public[0] subnet-abc123

# Import a resource that uses for_each
terraform import 'aws_iam_user.users["alice"]' alice
```

The `terraform import` command requires that a corresponding resource block already exists in your configuration. After import, run `terraform plan` and adjust the configuration until there are no planned changes.

### Import Blocks (Terraform 1.5+)

Terraform 1.5 introduced declarative import blocks — a major improvement over the CLI approach:

```hcl
# imports.tf — Declare imports alongside configuration
import {
  to = aws_vpc.main
  id = "vpc-0123456789abcdef0"
}

import {
  to = aws_s3_bucket.data
  id = "my-app-data-bucket"
}

import {
  to = aws_iam_role.lambda_exec
  id = "lambda-execution-role"
}
```

Benefits of import blocks:

- **Reviewable** — import declarations go through code review like any other change
- **Repeatable** — can be planned and applied as part of normal workflow
- **Generates configuration** — with `terraform plan -generate-config-out=generated.tf`, Terraform writes the resource configuration for you

```bash
# Generate configuration for imported resources
terraform plan -generate-config-out=generated_imports.tf

# Review the generated configuration, then apply
terraform apply
```

### Pulumi Import

Pulumi provides a similar import capability:

```bash
# Import an existing AWS S3 bucket
pulumi import aws:s3/bucket:Bucket my-bucket my-app-data-bucket

# Import and generate code
pulumi import aws:s3/bucket:Bucket my-bucket my-app-data-bucket --out import.ts
```

### CloudFormation Resource Import

CloudFormation supports importing existing resources into a stack:

```bash
# Create a change set of type IMPORT
aws cloudformation create-change-set \
  --stack-name my-stack \
  --change-set-name import-vpc \
  --change-set-type IMPORT \
  --resources-to-import "[{\"ResourceType\":\"AWS::EC2::VPC\",\"LogicalResourceId\":\"MainVPC\",\"ResourceIdentifier\":{\"VpcId\":\"vpc-0123456789abcdef0\"}}]" \
  --template-body file://template.yaml

# Execute the change set after review
aws cloudformation execute-change-set \
  --stack-name my-stack \
  --change-set-name import-vpc
```

---

## State Surgery

### When State Surgery is Needed

State surgery — directly manipulating the state file — is a last-resort operation needed when refactoring (renaming resources or moving them between modules), removing (detaching a resource from Terraform without destroying it), recovering (fixing corrupted state), or splitting (breaking a monolithic state into smaller states).

> **Always back up your state before performing state surgery.**

```bash
terraform state pull > state-backup-$(date +%Y%m%d-%H%M%S).json
```

### terraform state mv

Move a resource to a different address within the same state, or between states:

```bash
# Rename a resource
terraform state mv aws_instance.web aws_instance.app_server

# Move a resource into a module
terraform state mv aws_vpc.main module.networking.aws_vpc.this

# Move a resource from one module to another
terraform state mv module.old_network.aws_subnet.public module.new_network.aws_subnet.public

# Move a resource between state files
terraform state mv -state=source.tfstate -state-out=destination.tfstate \
  aws_instance.web aws_instance.web
```

For Terraform 1.1+, prefer `moved` blocks over `terraform state mv` — they are declarative and go through code review:

```hcl
# moved blocks in configuration — preferred over CLI state mv
moved {
  from = aws_instance.web
  to   = aws_instance.app_server
}

moved {
  from = aws_vpc.main
  to   = module.networking.aws_vpc.this
}
```

### terraform state rm

Remove a resource from state **without destroying** the actual infrastructure:

```bash
# Remove a single resource from state
terraform state rm aws_instance.legacy_server

# Remove a module and all its resources from state
terraform state rm module.deprecated_module

# Remove a specific instance from a count or for_each
terraform state rm 'aws_iam_user.users["alice"]'
```

Common use cases: migrating ownership to a different Terraform configuration, abandoning management of a resource, or fixing conflicts from resources deleted outside Terraform.

### terraform state pull and push

Pull and push allow direct manipulation of the raw state JSON:

```bash
# Pull current state to stdout
terraform state pull > current-state.json

# Push a modified state file back
terraform state push current-state.json

# Force push (override lineage/serial checks — DANGEROUS)
terraform state push -force modified-state.json
```

### Recovering Corrupted State

If state becomes corrupted, follow this recovery process:

```bash
# Step 1: Check if Terraform can read the state
terraform state list

# Step 2: If state is unreadable, restore from S3 versioning
aws s3api list-object-versions \
  --bucket mycompany-terraform-state \
  --prefix production/terraform.tfstate \
  --max-items 5

# Step 3: Download a previous version
aws s3api get-object \
  --bucket mycompany-terraform-state \
  --key production/terraform.tfstate \
  --version-id "abc123previousversion" \
  restored-state.json

# Step 4: Push the restored state and check for drift
terraform state push restored-state.json
terraform plan
```

For Pulumi, export and import state:

```bash
pulumi stack export > stack-backup.json
pulumi stack import < stack-backup.json
```

---

## State Migration

### Moving Between Backends

Migrating state between backends is a common operation when teams adopt remote state or switch providers:

```bash
# Example: Migrate from local to S3 backend
# Step 1: Add the S3 backend configuration to your .tf files
# Step 2: Run terraform init — Terraform detects the backend change
terraform init
# Terraform will ask: "Do you want to copy existing state to the new backend?"
# Answer: yes

# Step 3: Verify the migration
terraform state list
terraform plan  # Should show no changes
```

Migrating between two remote backends:

```bash
# Step 1: Pull state from current backend as a backup
terraform state pull > migration-backup.json

# Step 2: Update the backend configuration in .tf files

# Step 3: Reinitialize with migration
terraform init -migrate-state

# Step 4: Verify
terraform state list
terraform plan
```

### Splitting Monolithic State

As infrastructure grows, a single state file becomes unwieldy. Splitting into smaller, focused states improves performance and blast radius:

```
Before: Single state with 500+ resources (30s+ plan time)
After:  networking/ (~50 res) + compute/ (~30 res) + data/ (~20 res)
```

The splitting process:

```bash
# Step 1: Back up the current state
terraform state pull > monolith-backup.json

# Step 2: In the new networking/ directory, import resources
cd networking/ && terraform init
terraform import aws_vpc.main vpc-0123456789abcdef0
terraform import aws_subnet.public[0] subnet-aaa

# Step 3: Remove moved resources from the monolith
cd ../monolith/
terraform state rm aws_vpc.main
terraform state rm 'aws_subnet.public[0]'

# Step 4: Use data sources for cross-state references
```

```hcl
# In compute/main.tf — reference networking state
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "production/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "web" {
  subnet_id = data.terraform_remote_state.networking.outputs.public_subnet_ids[0]
}
```

### Merging States

Occasionally you need to merge two state files — for example, consolidating infrastructure that was managed separately:

```bash
# Step 1: Pull both state files
cd project-a/ && terraform state pull > state-a.json
cd ../project-b/ && terraform state pull > state-b.json

# Step 2: Move resources from project-b into project-a
cd ../project-a/
terraform state mv -state=../project-b/state-b.json -state-out=merge-temp.tfstate \
  aws_s3_bucket.data aws_s3_bucket.data

# Step 3: Verify
terraform state list
terraform plan
```

> **Note**: Merging state is error-prone. Prefer importing resources into the target state over direct state manipulation.

### Migration Checklist

Use this checklist for any state migration:

- [ ] Back up current state (`terraform state pull > backup.json`)
- [ ] Ensure no concurrent operations are running
- [ ] Test the migration in a non-production environment first
- [ ] Verify the target backend is properly configured and accessible
- [ ] Run `terraform plan` after migration — expect **no changes**
- [ ] Verify state locking works on the new backend
- [ ] Update CI/CD pipelines to use the new backend configuration
- [ ] Keep the backup for at least 30 days
- [ ] Remove old backend resources after confirming success

---

## Drift Detection and Reconciliation

### What is Drift

**Drift** occurs when the actual state of infrastructure diverges from what the IaC tool expects. Common causes include manual console changes, modifications by other tools or scripts, cloud provider default changes, and automated processes (auto-scaling, patching).

### Terraform Plan Drift Detection

Terraform detects drift during the `plan` phase by refreshing state against real infrastructure:

```bash
# Standard plan includes refresh (drift detection)
terraform plan

# Refresh-only mode — detect drift without planning config changes
terraform plan -refresh-only

# Apply refresh to update state without changing infrastructure
terraform apply -refresh-only
```

When drift is detected, Terraform reports the differences — for example, showing that an `instance_type` was changed from `t3.medium` to `t3.large` outside of Terraform.

### CloudFormation Drift Detection

CloudFormation provides built-in drift detection:

```bash
# Initiate drift detection on a stack
aws cloudformation detect-stack-drift \
  --stack-name my-stack

# Check drift detection status
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id "drift-detection-id"

# Get detailed drift results for each resource
aws cloudformation describe-stack-resource-drifts \
  --stack-name my-stack \
  --stack-resource-drift-status-filters MODIFIED DELETED
```

CloudFormation drift status types:

| Status | Meaning |
|---|---|
| `IN_SYNC` | Resource matches the template |
| `MODIFIED` | Resource properties differ from the template |
| `DELETED` | Resource was deleted outside CloudFormation |
| `NOT_CHECKED` | CloudFormation cannot check this resource type |

### Remediation Strategies

When drift is detected, you have several options:

| Strategy | When to Use | Action |
|---|---|---|
| **Accept config** | Manual change was a mistake | Run `terraform apply` to revert to configuration |
| **Accept reality** | Manual change was intentional | Update configuration to match, then `terraform apply` |
| **Refresh only** | State is stale but config is correct | Run `terraform apply -refresh-only` |
| **Ignore changes** | Resource is managed externally | Add `ignore_changes` lifecycle rule |
| **Prevent changes** | Resource must never be modified | Add `prevent_destroy` lifecycle rule |

```hcl
# Ignore external changes to specific attributes
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.medium"

  lifecycle {
    ignore_changes = [
      ami,          # AMI is updated by external patching process
      tags["UpdatedBy"],
    ]
  }
}

# Prevent accidental destruction
resource "aws_s3_bucket" "critical_data" {
  bucket = "critical-production-data"

  lifecycle {
    prevent_destroy = true
  }
}
```

---

## State Security

### Encrypting State

State files contain sensitive infrastructure details and often secrets. Encryption is critical at multiple layers:

1. **Encryption in Transit** — TLS/HTTPS for all API calls
2. **Encryption at Rest** — S3 SSE, Azure SSE, GCS default encryption
3. **Customer-Managed Keys** — KMS, Azure Key Vault, Cloud KMS
4. **Application-Level Encryption** — Pulumi passphrase, SOPS, Vault

### Access Control

Restrict who can read and write state using cloud IAM:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TerraformStateAccess",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::mycompany-terraform-state/*"
    },
    {
      "Sid": "TerraformStateLocking",
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:DeleteItem"],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/terraform-state-locks"
    },
    {
      "Sid": "TerraformStateKMS",
      "Effect": "Allow",
      "Action": ["kms:Decrypt", "kms:Encrypt", "kms:GenerateDataKey"],
      "Resource": "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012"
    }
  ]
}
```

Principle of least privilege for state access:

| Role | S3 Access | DynamoDB Access | KMS Access |
|---|---|---|---|
| **CI/CD pipeline** | GetObject, PutObject | GetItem, PutItem, DeleteItem | Decrypt, Encrypt, GenerateDataKey |
| **Developer (read-only)** | GetObject | GetItem | Decrypt |
| **Admin (break-glass)** | Full access | Full access | Full access |
| **Auditor** | GetObject (specific paths) | None | Decrypt |

### Audit Logging

Enable logging to track all state access:

```hcl
# S3 access logging
resource "aws_s3_bucket_logging" "state_logging" {
  bucket        = aws_s3_bucket.terraform_state.id
  target_bucket = aws_s3_bucket.access_logs.id
  target_prefix = "terraform-state-access/"
}
```

Additionally, enable AWS CloudTrail with S3 data events on the state bucket to capture API-level auditing of all `GetObject` and `PutObject` calls.

### Sensitive Values in State

Strategies to minimize sensitive data exposure in state:

| Strategy | Description | Trade-off |
|---|---|---|
| **External secret managers** | Use Vault, AWS Secrets Manager, or SSM Parameter Store | Secrets never enter state, but adds operational complexity |
| **Sensitive variables** | Mark variables as `sensitive` in Terraform | Masked in CLI output, but still in state JSON |
| **Pulumi secrets** | Pulumi encrypts secret values per-value in state | Strong encryption, but requires passphrase or KMS |
| **SOPS integration** | Encrypt secret files with SOPS, decrypt at apply time | Secrets are not stored in state if managed externally |

```hcl
# BAD — secret ends up in state
resource "aws_db_instance" "main" {
  engine         = "postgres"
  instance_class = "db.t3.medium"
  username       = "admin"
  password       = var.db_password  # Stored in state!
}

# BETTER — let AWS manage the password
resource "aws_db_instance" "main" {
  engine                      = "postgres"
  instance_class              = "db.t3.medium"
  username                    = "admin"
  manage_master_user_password = true  # AWS manages the password
}
```

---

## State at Scale

### Large State Files

As infrastructure grows, state files can become very large:

| Resources | Approximate State Size | Plan Time |
|---|---|---|
| 50 | ~100 KB | 5-10 seconds |
| 200 | ~500 KB | 15-30 seconds |
| 500 | ~2 MB | 30-60 seconds |
| 1,000 | ~5 MB | 1-3 minutes |
| 5,000+ | ~25 MB+ | 5-15+ minutes |

Large state files cause:

- **Slow plan/apply cycles** — every operation reads the entire state and refreshes resources
- **API rate limiting** — refresh queries every resource, hitting cloud API limits
- **Increased blast radius** — a mistake affects more resources
- **Lock contention** — longer operations hold locks longer, blocking other work

### Performance Implications

Terraform refreshes every resource in state during `plan` and `apply` by default:

```bash
# Target specific resources (use sparingly — debugging/emergencies only)
terraform plan -target=aws_instance.web

# Skip refresh when you know state is current
terraform plan -refresh=false

# Control concurrent API calls (default: 10)
terraform apply -parallelism=20
```

### State Splitting Strategies

Split state along natural boundaries:

```
Recommended State Boundaries:

  Foundation Layer (changes rarely, shared by all)
  ├── VPC, Subnets, Transit Gateway, IAM, DNS zones
  │
  ├── Platform Layer      Data Layer       Security Layer
  │   ├── EKS/ECS        ├── RDS          ├── WAF
  │   ├── ALB/NLB        ├── DynamoDB     ├── GuardDuty
  │   └── ECR            ├── S3           └── Config Rules
  │                      └── CloudFront
  │
  └── Application Layer (per-service, changes frequently)
      ├── Service A state
      ├── Service B state
      └── Service C state
```

Guidelines for splitting:

- **Rate of change** — group resources that change at similar frequencies
- **Ownership** — align state boundaries with team ownership
- **Blast radius** — limit the number of resources a single `apply` can affect
- **Dependencies** — minimize cross-state references (use outputs + remote state data sources)

---

## Multi-Account and Multi-Region State

### Patterns for Multi-Account State

Enterprise organizations use multiple AWS accounts (or Azure subscriptions, GCP projects) for isolation:

```
┌──────────────────────────────────────────────────────────┐
│           Multi-Account State Architecture                │
│                                                           │
│  ┌─────────────────┐  Central state backend               │
│  │  Management Acct │  S3 + DynamoDB + KMS                │
│  └────────┬─────────┘                                     │
│     ┌─────┼──────────────────┐                            │
│  ┌──▼──┐ ┌▼────┐ ┌──────┐ ┌─▼────────┐                   │
│  │ Dev │ │Stag │ │ Prod │ │ Shared   │                   │
│  └─────┘ └─────┘ └──────┘ └──────────┘                   │
│                                                           │
│  State paths:                                             │
│  s3://state/{account}/{region}/{component}/tfstate         │
└──────────────────────────────────────────────────────────┘
```

Configure backends with account-specific state paths:

```hcl
# Centralized state bucket in management account
terraform {
  backend "s3" {
    bucket         = "company-central-terraform-state"
    key            = "production/us-east-1/networking/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
    role_arn       = "arn:aws:iam::111111111111:role/TerraformStateAccess"
  }
}

# Assume role into the target account for resource management
provider "aws" {
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::222222222222:role/TerraformExecution"
  }
}
```

### Multi-Region State Architecture

For multi-region deployments, organize state by region:

```hcl
# State key convention: {account}/{region}/{component}/terraform.tfstate
# US East: key = "production/us-east-1/networking/terraform.tfstate"
# EU West: key = "production/eu-west-1/networking/terraform.tfstate"
# Note: The state bucket region stays the same — only the key changes
```

For Pulumi multi-region stacks:

```bash
# Create region-specific stacks
pulumi stack init production-us-east-1
pulumi stack init production-eu-west-1

# Set region-specific configuration per stack
pulumi stack select production-us-east-1
pulumi config set aws:region us-east-1
```

### Cross-Account State Access

Use `terraform_remote_state` data sources for cross-state references:

```hcl
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket   = "company-central-terraform-state"
    key      = "shared/networking/terraform.tfstate"
    region   = "us-east-1"
    role_arn = "arn:aws:iam::111111111111:role/TerraformStateReader"
  }
}

resource "aws_instance" "app" {
  subnet_id              = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
  vpc_security_group_ids = [data.terraform_remote_state.networking.outputs.app_security_group_id]
}
```

Alternatives to `terraform_remote_state`:

| Method | Pros | Cons |
|---|---|---|
| **terraform_remote_state** | Simple, built-in | Tight coupling, exposes all outputs |
| **SSM Parameter Store** | Loose coupling, simple key-value | AWS-specific |
| **Data sources** | Query real resources directly | No dependency tracking, slower |

---

## Next Steps

Continue your Infrastructure as Code learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | IaC Fundamentals | Core IaC concepts, declarative vs imperative, tool landscape |
| [01-TERRAFORM.md](01-TERRAFORM.md) | Terraform | HCL fundamentals, providers, modules, state management |
| [02-PULUMI.md](02-PULUMI.md) | Pulumi | General-purpose languages for IaC, stacks, automation API |
| [03-CLOUDFORMATION.md](03-CLOUDFORMATION.md) | AWS CloudFormation | AWS-native IaC, nested stacks, drift detection |
| [05-MODULES-AND-REUSE.md](05-MODULES-AND-REUSE.md) | Modules and Reuse | Module design, versioning, registries |
| [06-TESTING.md](06-TESTING.md) | IaC Testing | Policy as code, plan validation, integration testing |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Directory structure, naming, CI/CD integration |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial State Management documentation |
