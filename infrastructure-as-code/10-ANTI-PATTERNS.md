# Infrastructure as Code Anti-Patterns

A catalogue of the most common Infrastructure as Code mistakes — what they look like, why they are harmful, and exactly how to fix them. Use this document as a code-review checklist, a pre-production gate, or a team learning resource.

---

## Table of Contents

- [Introduction](#introduction)
- [Anti-Patterns Summary Table](#anti-patterns-summary-table)
- [1. ClickOps (Manual Console Changes)](#1-clickops-manual-console-changes)
- [2. Monolithic State Files](#2-monolithic-state-files)
- [3. Hardcoded Values](#3-hardcoded-values)
- [4. Secrets in Code or State](#4-secrets-in-code-or-state)
- [5. No Remote State Backend](#5-no-remote-state-backend)
- [6. Ignoring Drift](#6-ignoring-drift)
- [7. Copy-Paste Instead of Modules](#7-copy-paste-instead-of-modules)
- [8. No CI/CD for Infrastructure](#8-no-cicd-for-infrastructure)
- [9. God Modules](#9-god-modules)
- [10. Terraform Apply Without Plan Review](#10-terraform-apply-without-plan-review)
- [11. Missing Resource Tagging](#11-missing-resource-tagging)
- [12. Over-Engineering Abstractions](#12-over-engineering-abstractions)
- [Quick Reference Checklist](#quick-reference-checklist)
- [Next Steps](#next-steps)
- [Version History](#version-history)

---

## Introduction

### Why Anti-Patterns Matter

Anti-patterns are recurring practices that seem reasonable at first glance but create significant problems over time. In Infrastructure as Code, the consequences of anti-patterns are typically felt during production incidents, scaling efforts, security audits, or cloud cost reviews — the exact moments when your infrastructure needs to be reliable, auditable, and reproducible.

The patterns documented here represent real failures observed across production IaC environments. Each one is:

- **Seductive** — it felt like the right approach when first implemented
- **Harmful** — it creates infrastructure drift, security vulnerabilities, state corruption, or operational overhead
- **Fixable** — there is a well-understood better approach

### How to Use This Document

1. **Pre-production review:** Use the [Quick Reference Checklist](#quick-reference-checklist) before promoting infrastructure code to production
2. **Code review:** Reference specific sections when reviewing Terraform, Pulumi, or CloudFormation PRs
3. **Incident retros:** After an infrastructure incident, check which anti-patterns contributed to the failure
4. **Team onboarding:** Assign this document to new engineers before they write production infrastructure code
5. **Periodic audit:** Run through the checklist quarterly for existing infrastructure configurations

---

## Anti-Patterns Summary Table

| # | Anti-Pattern | Category | Severity |
|---|--------------|----------|----------|
| 1 | ClickOps (Manual Console Changes) | Reliability | 🔴 Critical |
| 2 | Monolithic State Files | Scalability | 🔴 Critical |
| 3 | Hardcoded Values | Maintainability | 🔴 Critical |
| 4 | Secrets in Code or State | Security | 🔴 Critical |
| 5 | No Remote State Backend | Reliability | 🔴 Critical |
| 6 | Ignoring Drift | Consistency | 🟠 High |
| 7 | Copy-Paste Instead of Modules | Maintainability | 🟠 High |
| 8 | No CI/CD for Infrastructure | Reliability | 🟠 High |
| 9 | God Modules | Complexity | 🟠 High |
| 10 | Terraform Apply Without Plan Review | Safety | 🟠 High |
| 11 | Missing Resource Tagging | Operations | 🟡 Medium |
| 12 | Over-Engineering Abstractions | Complexity | 🟡 Medium |

---

## 1. ClickOps (Manual Console Changes)

Making infrastructure changes by clicking through the AWS Console, Azure Portal, or GCP Console instead of writing code. The change exists in the cloud provider but nowhere in version control. Nobody knows what was changed, when, or why — until something breaks and the person who clicked is on vacation.

### Why It Happens

- A "quick fix" is needed during an incident and the console feels faster
- Engineers are unfamiliar with the IaC tool and fall back to what they know
- Management pressure to deliver immediately without "wasting time" writing code
- Prototyping in the console and forgetting to codify the result

### What Goes Wrong

```
Engineer A: Creates an S3 bucket via the console for a "quick demo"
Engineer B: Runs terraform apply — Terraform has no knowledge of the bucket
Engineer A: Adds a lifecycle policy via the console
Engineer C: Writes Terraform for a "new" bucket with the same name

$ terraform apply
╷
│ Error: creating Amazon S3 Bucket (demo-data-bucket):
│        BucketAlreadyExists: The requested bucket name is not available.
│
│   with aws_s3_bucket.demo_data,
│   on main.tf line 12, in resource "aws_s3_bucket" "demo_data":
│   12: resource "aws_s3_bucket" "demo_data" {
╵

Nobody knows who created it, what IAM policies are attached, or whether it
holds production data. Deleting it might cause an outage. Importing it
requires reverse-engineering every attribute.
```

```
Console changes vs. code-managed infrastructure:

   Console (ClickOps)              Code (IaC)
   ─────────────────               ──────────
   ❌ No audit trail               ✅ Git history
   ❌ No peer review               ✅ Pull request review
   ❌ No reproducibility           ✅ Runs identically every time
   ❌ No rollback                  ✅ git revert + apply
   ❌ Single point of knowledge    ✅ Shared codebase
   ❌ Drift from desired state     ✅ Drift detection built in
```

### The Fix

❌ **Bad — manual console change:**

```
1. Log in to AWS Console
2. Navigate to EC2 → Security Groups
3. Click "Create Security Group"
4. Add inbound rule: port 443, source 0.0.0.0/0
5. Click "Create"
6. Hope you remember to tell the team
```

✅ **Good — every change is code:**

```hcl
# security-groups.tf — reviewed, versioned, auditable
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Allow HTTPS traffic"
  vpc_id      = var.vpc_id

  ingress {
    description = "HTTPS from internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "web-sg"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

✅ **Good — enforce no-console-changes with an SCP:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyConsoleChanges",
      "Effect": "Deny",
      "Action": [
        "ec2:RunInstances",
        "ec2:CreateSecurityGroup",
        "s3:CreateBucket",
        "rds:CreateDBInstance"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:role/TerraformExecutionRole"
        }
      }
    }
  ]
}
```

### Key Takeaway

If a change is not in code, it does not exist. Enforce IaC-only changes with Service Control Policies and treat console changes as incidents, not shortcuts.

---

## 2. Monolithic State Files

Storing all infrastructure for an entire organisation — networking, databases, compute, DNS, IAM — in a single Terraform state file. Every `terraform plan` takes ten minutes, every `apply` risks collateral damage across unrelated systems, and state locking blocks every other team.

### Why It Happens

- The project started small and grew without restructuring
- Teams do not understand state file boundaries or workspaces
- "It works, so why split it?" — until it doesn't
- Fear of cross-state references and `terraform_remote_state` data sources

### What Goes Wrong

```
Monolithic state — everything in one file:

┌─────────────────────────────────────────────────────┐
│                   terraform.tfstate                  │
│                                                     │
│  VPC · Subnets · Route Tables · NAT Gateways        │
│  RDS Clusters · ElastiCache · DynamoDB Tables        │
│  ECS Services · ALBs · Target Groups                 │
│  IAM Roles · Policies · Users                        │
│  S3 Buckets · CloudFront Distributions               │
│  Route53 Zones · ACM Certificates                    │
│  Lambda Functions · API Gateways                     │
│  ... 847 resources, 14 MB state file ...             │
│                                                     │
│  terraform plan: 9 minutes 47 seconds                │
│  State lock held by: entire duration                 │
│  Blast radius: everything                            │
└─────────────────────────────────────────────────────┘

Team A needs to update an ECS task definition — blocked.
Team B needs to add a DNS record — blocked.
Team C accidentally deletes the wrong resource — blast radius is everything.
```

### The Fix

❌ **Bad — one state file for everything:**

```hcl
# main.tf — 3,000 lines, 847 resources, one state file
resource "aws_vpc" "main" { ... }
resource "aws_subnet" "public" { ... }
resource "aws_rds_cluster" "primary" { ... }
resource "aws_ecs_service" "api" { ... }
resource "aws_iam_role" "lambda" { ... }
resource "aws_s3_bucket" "assets" { ... }
# ... 841 more resources ...
```

✅ **Good — split by domain with independent state:**

```
infra/
├── networking/          # State: networking.tfstate
│   ├── main.tf          # VPC, subnets, route tables, NAT
│   ├── variables.tf
│   ├── outputs.tf       # Exports VPC ID, subnet IDs
│   └── backend.tf
├── data-stores/         # State: data-stores.tfstate
│   ├── main.tf          # RDS, ElastiCache, DynamoDB
│   ├── variables.tf
│   ├── outputs.tf       # Exports connection strings
│   └── backend.tf
├── services/
│   ├── api/             # State: services-api.tfstate
│   │   ├── main.tf      # ECS service, ALB, target group
│   │   └── backend.tf
│   └── worker/          # State: services-worker.tfstate
│       ├── main.tf
│       └── backend.tf
└── identity/            # State: identity.tfstate
    ├── main.tf          # IAM roles, policies
    └── backend.tf
```

```hcl
# networking/backend.tf — isolated state per domain
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "networking/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

```hcl
# services/api/main.tf — reads outputs from networking state
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_ecs_service" "api" {
  name            = "api"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.api.arn

  network_configuration {
    subnets = data.terraform_remote_state.networking.outputs.private_subnet_ids
  }
}
```

### Key Takeaway

Split state by team ownership and change frequency. Networking changes monthly, application services change daily — they should never share a state file or a blast radius.

---

## 3. Hardcoded Values

Embedding environment-specific values — region names, instance sizes, CIDR blocks, account IDs — directly in resource definitions instead of using variables. The same infrastructure cannot be deployed to a second environment without find-and-replace surgery across dozens of files.

### Why It Happens

- "We only have one environment" (until staging is needed next week)
- Copy-paste from documentation or tutorials that use hardcoded values
- Variables feel like unnecessary complexity for a small project
- No enforcement mechanism in code review

### What Goes Wrong

```
$ grep -rn "us-east-1" infra/
infra/main.tf:3:  region = "us-east-1"
infra/rds.tf:7:   availability_zone = "us-east-1a"
infra/s3.tf:12:   bucket = "mycompany-prod-us-east-1-assets"
infra/vpc.tf:5:   cidr_block = "10.0.0.0/16"
infra/ec2.tf:9:   instance_type = "t3.large"
infra/ec2.tf:10:  ami = "ami-0abcdef1234567890"

Need to deploy to eu-west-1 for GDPR?
→ Change 47 files, hope you didn't miss one, pray nothing breaks.
```

### The Fix

❌ **Bad — values baked into resources:**

```hcl
# main.tf — hardcoded everywhere
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.large"
  subnet_id     = "subnet-0123456789abcdef0"

  tags = {
    Name = "prod-web-server"
  }
}

resource "aws_db_instance" "main" {
  engine               = "postgres"
  instance_class       = "db.r5.xlarge"
  allocated_storage    = 100
  db_name              = "proddb"
  username             = "admin"
  password             = "SuperSecret123!"
  skip_final_snapshot  = true
}
```

✅ **Good — parameterised with variables and tfvars:**

```hcl
# variables.tf — single source of truth for configuration
variable "region" {
  description = "AWS region for all resources"
  type        = string
}

variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.medium"
}

variable "db_instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.r5.large"
}
```

```hcl
# environments/prod.tfvars
region            = "us-east-1"
environment       = "prod"
instance_type     = "t3.large"
db_instance_class = "db.r5.xlarge"

# environments/staging.tfvars
region            = "us-east-1"
environment       = "staging"
instance_type     = "t3.medium"
db_instance_class = "db.r5.large"

# environments/eu-prod.tfvars — new region in minutes, not days
region            = "eu-west-1"
environment       = "prod"
instance_type     = "t3.large"
db_instance_class = "db.r5.xlarge"
```

```hcl
# main.tf — clean, parameterised, reusable
provider "aws" {
  region = var.region
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  subnet_id     = module.networking.private_subnet_ids[0]

  tags = {
    Name        = "${var.environment}-web-server"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

### Key Takeaway

If a value could ever differ between environments, it must be a variable. Hardcoded values are the number one reason teams cannot reuse their infrastructure code.

---

## 4. Secrets in Code or State

Storing passwords, API keys, database credentials, or TLS certificates directly in `.tf` files, `terraform.tfvars`, or allowing them to persist in plain text inside the Terraform state file. A single `git log` or S3 bucket misconfiguration exposes every secret to anyone with repository or bucket access.

### Why It Happens

- Tutorials show `password = "mysecret"` in examples and engineers copy them
- State files are stored in S3 without encryption, and nobody audits their contents
- Engineers do not realise that any value passed to a Terraform resource ends up in state
- "It's a private repo — nobody will see it"

### What Goes Wrong

```
$ git log --all -p -- '*.tf' '*.tfvars' | grep -i "password\|secret\|api_key"
+  password             = "SuperSecret123!"
+  master_password      = "db-admin-2024!"
+  api_key              = "sk-live-abc123xyz789"

$ aws s3 cp s3://terraform-state/prod/terraform.tfstate - | jq '.resources[]
  | select(.type == "aws_db_instance") | .instances[].attributes.password'
"db-admin-2024!"

Every secret is in plain text in:
  1. Git history (forever, even after deletion)
  2. State file (readable by anyone with bucket access)
  3. Plan output (printed to CI logs)
```

### The Fix

❌ **Bad — secrets in code and state:**

```hcl
# DO NOT DO THIS — secrets in plain text
resource "aws_db_instance" "main" {
  engine         = "postgres"
  instance_class = "db.r5.large"
  db_name        = "appdb"
  username       = "admin"
  password       = "SuperSecret123!"  # In git history forever
}

variable "api_key" {
  default = "sk-live-abc123xyz789"  # In git history forever
}
```

✅ **Good — secrets from a vault, never in code:**

```hcl
# Retrieve secrets from AWS Secrets Manager at apply time
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/database/master-password"
}

resource "aws_db_instance" "main" {
  engine         = "postgres"
  instance_class = var.db_instance_class
  db_name        = "appdb"
  username       = "admin"
  password       = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

✅ **Good — Pulumi with encrypted secrets:**

```typescript
// Pulumi encrypts secrets in state automatically
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const config = new pulumi.Config();
const dbPassword = config.requireSecret("dbPassword"); // encrypted in state

const db = new aws.rds.Instance("main", {
    engine: "postgres",
    instanceClass: "db.r5.large",
    dbName: "appdb",
    username: "admin",
    password: dbPassword, // never stored in plain text
});
```

✅ **Good — encrypt the state backend:**

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:111111111111:key/abcd-1234"
    dynamodb_table = "terraform-locks"
  }
}
```

### Key Takeaway

Secrets must never appear in `.tf` files, `.tfvars` files, or CI logs. Use a secrets manager, encrypt your state backend, and mark sensitive outputs with `sensitive = true`.

---

## 5. No Remote State Backend

Storing `terraform.tfstate` on a developer's local filesystem instead of a shared, locked, encrypted remote backend. Two engineers run `terraform apply` at the same time and corrupt the state. A laptop is lost and the state file — the only record of what Terraform manages — is gone forever.

### Why It Happens

- Terraform defaults to local state and tutorials rarely mention remote backends
- "I'm the only one working on this" (until someone else joins the team)
- Setting up S3 + DynamoDB or Terraform Cloud feels like unnecessary overhead for a small project
- Engineers do not understand that the state file is the single source of truth

### What Goes Wrong

```
Local state disaster timeline:

Day 1:   Alice creates infra, state is on her laptop
Day 5:   Bob clones the repo — no state file (it's in .gitignore)
Day 5:   Bob runs terraform apply — Terraform creates DUPLICATE resources
Day 6:   Two identical VPCs, two RDS instances, double the cost
Day 10:  Alice's laptop hard drive fails — state file is gone
Day 10:  Terraform has no idea what it manages — manual cleanup required

         Alice's Laptop              Bob's Laptop
         ┌──────────────┐            ┌──────────────┐
         │ tfstate v3   │            │ tfstate v1   │
         │ (10 resources)│           │ (4 resources) │
         └──────┬───────┘            └──────┬───────┘
                │                           │
                ▼                           ▼
         ┌──────────────────────────────────────────┐
         │              AWS Account                  │
         │  14 resources (some duplicated, some      │
         │  orphaned, nobody knows which is which)   │
         └──────────────────────────────────────────┘
```

### The Fix

❌ **Bad — local state with no locking:**

```hcl
# No backend block = local file terraform.tfstate
# No locking, no encryption, no sharing, no backup
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

✅ **Good — remote state with locking and encryption:**

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/networking/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:111111111111:key/abcd-1234"
    dynamodb_table = "terraform-locks"
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

✅ **Good — bootstrap the backend with a separate config:**

```hcl
# state-backend/main.tf — create the backend resources first
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state"

  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name      = "Terraform State"
    ManagedBy = "terraform"
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
      kms_master_key_id = aws_kms_key.terraform.arn
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name      = "Terraform Lock Table"
    ManagedBy = "terraform"
  }
}
```

### Key Takeaway

Remote state with locking is not optional — it is the foundation of safe, collaborative IaC. Set it up before writing a single resource, not after the first state corruption incident.

---

## 6. Ignoring Drift

Never running `terraform plan` to detect differences between the declared configuration and the actual cloud state. Resources are modified outside of Terraform — via the console, CLI scripts, or other tools — and the drift silently accumulates until the next `terraform apply` overwrites the manual changes or fails in unexpected ways.

### Why It Happens

- No scheduled drift detection — teams only run `plan` when making changes
- Console changes (ClickOps) introduce drift that nobody knows about
- Auto-scaling, cloud provider maintenance, or other automation modifies resources
- Teams treat `terraform apply` as a one-time setup tool, not a continuous reconciliation loop

### What Goes Wrong

```
Drift accumulation over time:

Week 1:  Terraform state and cloud match perfectly
Week 2:  Someone adds a security group rule via console (drift: 1 resource)
Week 3:  Auto-scaling adds tags to instances (drift: 3 resources)
Week 5:  DBA changes RDS parameter group via CLI (drift: 5 resources)
Week 8:  Engineer runs terraform plan for a "small change":

$ terraform plan

~ aws_security_group.web
    ~ ingress: [CHANGED — rule added outside Terraform, will be REMOVED]

~ aws_db_instance.main
    ~ parameter_group_name: "custom-pg" → "default.postgres15"
    [REVERT — DBA's performance tuning will be destroyed]

~ aws_instance.web[0]
    ~ tags: { AutoScaling: "true" } → {} [REMOVED]

Plan: 0 to add, 5 to change, 0 to destroy.

The "small change" now has 5 unexpected modifications.
Engineer applies anyway → production database performance tanks.
```

### The Fix

❌ **Bad — no drift detection:**

```yaml
# CI pipeline only runs plan/apply on PR merge
# Drift accumulates silently between deployments
name: Terraform Deploy
on:
  push:
    branches: [main]
jobs:
  apply:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: terraform init
      - run: terraform apply -auto-approve
```

✅ **Good — scheduled drift detection with alerts:**

```yaml
# .github/workflows/drift-detection.yml
name: Terraform Drift Detection
on:
  schedule:
    - cron: "0 8 * * 1-5"  # Every weekday at 8 AM

jobs:
  detect-drift:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        stack: [networking, data-stores, services/api, identity]
    steps:
      - uses: actions/checkout@v4

      - name: Terraform Init
        working-directory: infra/${{ matrix.stack }}
        run: terraform init -input=false

      - name: Detect Drift
        id: plan
        working-directory: infra/${{ matrix.stack }}
        run: |
          terraform plan -detailed-exitcode -input=false -out=plan.out 2>&1 | tee plan.txt
          echo "exitcode=$?" >> "$GITHUB_OUTPUT"
        continue-on-error: true

      - name: Alert on Drift
        if: steps.plan.outputs.exitcode == '2'
        run: |
          echo "🚨 Drift detected in ${{ matrix.stack }}"
          # Send Slack notification or create GitHub issue
```

✅ **Good — use lifecycle rules to handle expected changes:**

```hcl
resource "aws_autoscaling_group" "web" {
  name                = "${var.environment}-web-asg"
  min_size            = var.asg_min_size
  max_size            = var.asg_max_size
  desired_capacity    = var.asg_desired_capacity
  vpc_zone_identifier = var.private_subnet_ids

  lifecycle {
    ignore_changes = [desired_capacity]  # Auto-scaling changes this
  }
}
```

### Key Takeaway

Drift is not a possibility — it is a certainty. Run `terraform plan` on a schedule, alert on unexpected changes, and use `lifecycle.ignore_changes` only for values that are legitimately managed outside Terraform.

---

## 7. Copy-Paste Instead of Modules

Duplicating entire blocks of Terraform code across environments or projects instead of extracting reusable modules. A security fix must be applied to 14 copies of the same VPC configuration, and inevitably 3 of them get missed.

### Why It Happens

- Modules feel like over-engineering for a small project
- Engineers are not confident writing modules with proper input/output contracts
- "Each environment is slightly different" (usually by one or two variables)
- Copy-paste is faster in the short term

### What Goes Wrong

```
Copy-paste sprawl:

environments/
├── dev/
│   └── main.tf          # 200 lines — VPC + subnets + NAT + routes
├── staging/
│   └── main.tf          # 200 lines — same code, different CIDR
├── prod-us/
│   └── main.tf          # 200 lines — same code, different CIDR
├── prod-eu/
│   └── main.tf          # 200 lines — same code, different CIDR + region
└── disaster-recovery/
    └── main.tf          # 200 lines — same code, different everything

Security team: "Add VPC flow logs to all VPCs"
→ Modify 5 files, hope they all get the same change
→ PR reviewer must verify 5 identical diffs
→ 3 months later: DR environment still missing flow logs
```

### The Fix

❌ **Bad — copy-pasted VPC definition in every environment:**

```hcl
# environments/dev/main.tf — duplicated in 5 places
resource "aws_vpc" "main" {
  cidr_block           = "10.1.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = "dev-vpc" }
}

resource "aws_subnet" "public_a" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.1.1.0/24"
  availability_zone = "us-east-1a"
  tags = { Name = "dev-public-a" }
}

resource "aws_subnet" "public_b" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.1.2.0/24"
  availability_zone = "us-east-1b"
  tags = { Name = "dev-public-b" }
}

# ... 150 more lines of subnets, NAT, routes, NACLs ...
```

✅ **Good — a reusable module called once per environment:**

```hcl
# modules/networking/main.tf — single source of truth
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(var.common_tags, {
    Name = "${var.environment}-vpc"
  })
}

resource "aws_flow_log" "vpc" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_log.arn
  log_destination = aws_cloudwatch_log_group.flow_log.arn
}

# ... subnets, NAT, routes — defined once, parameterised ...
```

```hcl
# modules/networking/variables.tf
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "common_tags" {
  description = "Tags applied to all resources"
  type        = map(string)
  default     = {}
}
```

```hcl
# environments/dev/main.tf — 8 lines instead of 200
module "networking" {
  source      = "../../modules/networking"
  vpc_cidr    = "10.1.0.0/16"
  environment = "dev"
  common_tags = { ManagedBy = "terraform", Team = "platform" }
}

# environments/prod-us/main.tf — same module, different inputs
module "networking" {
  source      = "../../modules/networking"
  vpc_cidr    = "10.10.0.0/16"
  environment = "prod"
  common_tags = { ManagedBy = "terraform", Team = "platform" }
}
```

### Key Takeaway

If you copy-paste a block of Terraform more than once, extract it into a module. One source of truth means one place to fix, one place to review, and one place to add security controls.

---

## 8. No CI/CD for Infrastructure

Running `terraform apply` from a developer's laptop instead of through an automated pipeline. There is no audit trail of who applied what, no peer review of the plan output, and no consistent environment for execution. Two engineers with different Terraform versions or provider versions produce different results.

### Why It Happens

- "Setting up CI/CD for Terraform is hard" — it requires state access and cloud credentials
- Teams do not know how to safely pass `terraform plan` output to `terraform apply` in CI
- Fear of automated infrastructure changes — "I want to see the plan before applying"
- Local apply feels faster during development

### What Goes Wrong

```
Local apply problems:

Engineer A (Terraform 1.6, AWS provider 5.30):
  $ terraform apply    → creates resources with provider 5.30 schema

Engineer B (Terraform 1.8, AWS provider 5.55):
  $ terraform apply    → state is now managed by different versions
  $ terraform plan     → shows unexpected changes due to provider drift

Engineer C (on VPN, expired credentials):
  $ terraform apply    → partial apply, 3 of 7 resources created
  $ terraform apply    → retry fails with state lock (Engineer A is also running)

No record of who applied what, when, or why.
No plan review — engineers apply directly.
No consistent provider versions across team members.
```

### The Fix

❌ **Bad — apply from laptop:**

```bash
# Every engineer runs from their own machine
$ cd infra/
$ terraform init
$ terraform plan      # maybe
$ terraform apply     # YOLO
```

✅ **Good — plan on PR, apply on merge:**

```yaml
# .github/workflows/terraform.yml
name: Terraform
on:
  pull_request:
    paths: ["infra/**"]
  push:
    branches: [main]
    paths: ["infra/**"]

permissions:
  contents: read
  pull-requests: write
  id-token: write

jobs:
  plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.8.0"

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/TerraformPlanRole
          aws-region: us-east-1

      - name: Terraform Init
        run: terraform -chdir=infra init -input=false

      - name: Terraform Plan
        id: plan
        run: terraform -chdir=infra plan -input=false -no-color -out=plan.out

      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan 📖
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            *Pushed by: @${{ github.actor }}*`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

  apply:
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.8.0"

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/TerraformApplyRole
          aws-region: us-east-1

      - name: Terraform Init
        run: terraform -chdir=infra init -input=false

      - name: Terraform Apply
        run: terraform -chdir=infra apply -auto-approve -input=false
```

### Key Takeaway

Infrastructure changes must flow through the same PR → review → merge → deploy cycle as application code. Pin Terraform and provider versions in CI, and never grant `apply` credentials to developer laptops.

---

## 9. God Modules

Creating a single Terraform module that provisions an entire application stack — VPC, subnets, security groups, load balancers, ECS clusters, RDS databases, DNS records, and certificates — all controlled by dozens of boolean flags and conditional logic. The module is impossible to understand, test, or reuse for any purpose other than the original one.

### Why It Happens

- "One module to rule them all" feels elegant and DRY
- Engineers conflate "reuse" with "do everything" — they add features instead of composing modules
- Fear of managing multiple modules and their interdependencies
- Organic growth without refactoring — flags are added incrementally

### What Goes Wrong

```
God module interface:

module "application" {
  source = "./modules/everything"

  # 67 input variables
  create_vpc              = true
  create_subnets          = true
  create_nat              = var.environment == "prod" ? true : false
  create_rds              = true
  create_redis            = var.enable_caching
  create_ecs              = true
  create_fargate          = false
  create_ec2              = true
  create_alb              = true
  create_nlb              = false
  create_cloudfront       = var.environment == "prod"
  create_waf              = var.environment == "prod"
  enable_multi_az         = var.environment == "prod"
  enable_encryption       = true
  enable_backups          = var.environment != "dev"
  enable_monitoring       = true
  enable_logging          = true
  rds_engine              = "postgres"
  rds_multi_az            = var.environment == "prod"
  # ... 48 more variables ...
}

Inside the module: 2,400 lines of HCL with nested count/for_each
conditionals that nobody can follow.
```

### The Fix

❌ **Bad — one module that does everything:**

```hcl
# modules/everything/main.tf — 2,400 lines
resource "aws_vpc" "main" {
  count      = var.create_vpc ? 1 : 0
  cidr_block = var.vpc_cidr
  # ...
}

resource "aws_db_instance" "main" {
  count          = var.create_rds ? 1 : 0
  engine         = var.rds_engine
  instance_class = var.create_rds && var.enable_multi_az ? var.rds_multi_az_class : var.rds_class
  multi_az       = var.create_rds && var.enable_multi_az ? true : false
  # ... 50 more lines of conditional logic ...
}

resource "aws_ecs_service" "main" {
  count = var.create_ecs && !var.create_fargate ? 1 : 0
  # ...
  network_configuration {
    subnets = var.create_vpc ? aws_subnet.private[*].id : var.existing_subnet_ids
  }
}
```

✅ **Good — small, composable modules:**

```hcl
# Compose small modules — each one does one thing well
module "networking" {
  source      = "./modules/networking"
  vpc_cidr    = "10.0.0.0/16"
  environment = var.environment
  az_count    = 3
}

module "database" {
  source         = "./modules/rds-postgres"
  environment    = var.environment
  subnet_ids     = module.networking.private_subnet_ids
  vpc_id         = module.networking.vpc_id
  instance_class = var.db_instance_class
  multi_az       = var.environment == "prod"
}

module "service" {
  source          = "./modules/ecs-service"
  environment     = var.environment
  subnet_ids      = module.networking.private_subnet_ids
  vpc_id          = module.networking.vpc_id
  lb_target_group = module.load_balancer.target_group_arn
  db_endpoint     = module.database.endpoint
}

module "load_balancer" {
  source     = "./modules/alb"
  vpc_id     = module.networking.vpc_id
  subnet_ids = module.networking.public_subnet_ids
}
```

```
Composition vs. God Module:

God Module:                        Composable Modules:
┌────────────────────┐             ┌──────────┐  ┌──────────┐
│   "everything"     │             │networking│──│ database │
│                    │             └──────────┘  └──────────┘
│  VPC + Subnets     │                  │              │
│  RDS + Redis       │             ┌──────────┐  ┌──────────┐
│  ECS + ALB         │             │  service │──│   ALB    │
│  CloudFront + WAF  │             └──────────┘  └──────────┘
│  IAM + KMS         │
│  67 variables      │             Each module: < 200 lines
│  2,400 lines       │             Clear inputs and outputs
│  Untestable        │             Independently testable
└────────────────────┘             Reusable across projects
```

### Key Takeaway

A good module does one thing, has fewer than 15 input variables, and can be understood in 5 minutes. If your module needs a boolean flag to decide whether to create a resource, it should be two modules.

---

## 10. Terraform Apply Without Plan Review

Running `terraform apply` without first reviewing the plan output, or using `terraform apply -auto-approve` in production without human verification. A resource rename that looks harmless in code triggers a destroy-and-recreate cycle that takes down a production database.

### Why It Happens

- Engineers trust that their small code change will produce a small plan
- CI/CD pipelines use `-auto-approve` to avoid manual intervention
- Plan output is too verbose — engineers scroll past hundreds of unchanged resources
- Time pressure during incidents leads to skipping the review step

### What Goes Wrong

```
$ terraform plan

  # aws_db_instance.main must be replaced
  # (imported from "mydb" which does not match "main")
-/+ resource "aws_db_instance" "main" {
      ~ identifier = "mydb" -> "main"   # forces replacement
        # (this destroys the database and creates a new empty one)
      ...
    }

Plan: 1 to add, 0 to change, 1 to destroy.

Engineer sees "1 to add, 1 to destroy" and thinks it's a rename.
Actually: production database deleted, all data lost.
```

### The Fix

❌ **Bad — auto-approve in production:**

```yaml
# Dangerous — no human reviews the plan before apply
- name: Terraform Apply
  run: terraform apply -auto-approve
```

❌ **Bad — plan and apply in one step:**

```bash
# No chance to review what will happen
$ terraform apply
```

✅ **Good — plan saved to file, reviewed, then applied:**

```bash
# Step 1: Generate plan and save to file
$ terraform plan -out=tfplan

# Step 2: Review the plan (or post to PR for team review)
$ terraform show tfplan

# Step 3: Apply only the reviewed plan
$ terraform apply tfplan
```

✅ **Good — CI posts plan to PR, apply requires approval:**

```yaml
# Plan is posted as a PR comment for the team to review
- name: Terraform Plan
  id: plan
  run: |
    terraform plan -input=false -no-color -out=tfplan 2>&1 | tee plan.txt
    echo "## Terraform Plan" >> "$GITHUB_STEP_SUMMARY"
    echo '```' >> "$GITHUB_STEP_SUMMARY"
    cat plan.txt >> "$GITHUB_STEP_SUMMARY"
    echo '```' >> "$GITHUB_STEP_SUMMARY"

# Apply only runs after merge AND requires environment approval
- name: Terraform Apply
  if: github.ref == 'refs/heads/main'
  run: terraform apply -auto-approve -input=false
  environment: production  # requires manual approval in GitHub
```

✅ **Good — protect critical resources from accidental deletion:**

```hcl
resource "aws_db_instance" "main" {
  identifier     = "prod-database"
  engine         = "postgres"
  instance_class = var.db_instance_class

  lifecycle {
    prevent_destroy = true  # Terraform will refuse to destroy this
  }
}
```

### Key Takeaway

Never apply without reviewing the plan. Use `-out=tfplan` to save plans, post them to PRs for review, and protect critical resources with `lifecycle { prevent_destroy = true }`.

---

## 11. Missing Resource Tagging

Deploying cloud resources without consistent tags for ownership, environment, cost centre, or project. When the AWS bill arrives, nobody knows which team owns the $12,000 NAT Gateway. When a security audit runs, nobody knows which resources belong to which application.

### Why It Happens

- Tags are not required by the cloud provider — resources work without them
- No organisational tagging policy or enforcement mechanism
- Engineers forget to add tags, and code review does not catch it
- Different teams use different tag keys (`env` vs `environment` vs `Environment`)

### What Goes Wrong

```
$ aws ce get-cost-and-usage --granularity MONTHLY --metrics UnblendedCost \
    --group-by Type=TAG,Key=Team

{
  "ResultsByTime": [{
    "Groups": [
      { "Keys": ["Team$platform"],  "Metrics": { "UnblendedCost": "$4,200" }},
      { "Keys": ["Team$"],          "Metrics": { "UnblendedCost": "$28,700" }}
    ]
  }]
}

$28,700/month in UNTAGGED resources — no one knows who owns them.
Cannot identify which resources belong to decommissioned projects.
Cannot enforce cost budgets per team or environment.
```

### The Fix

❌ **Bad — no tags or inconsistent tags:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.large"
  # No tags — invisible in cost reports, audits, and incident triage
}

resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
  tags   = { env = "production" }  # inconsistent key name
}
```

✅ **Good — enforce tags with default_tags and validation:**

```hcl
# provider.tf — default tags applied to every resource automatically
provider "aws" {
  region = var.region

  default_tags {
    tags = {
      Environment = var.environment
      Team        = var.team
      Project     = var.project
      ManagedBy   = "terraform"
      CostCentre  = var.cost_centre
      Repository  = "github.com/mycompany/infra"
    }
  }
}
```

```hcl
# variables.tf — required tags with validation
variable "environment" {
  description = "Deployment environment"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "team" {
  description = "Owning team"
  type        = string
  validation {
    condition     = length(var.team) > 0
    error_message = "Team tag is required."
  }
}

variable "cost_centre" {
  description = "Cost centre code for billing"
  type        = string
  validation {
    condition     = can(regex("^CC-[0-9]{4}$", var.cost_centre))
    error_message = "Cost centre must match format CC-XXXX."
  }
}
```

✅ **Good — enforce tagging with AWS SCP or config rules:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUntaggedResources",
      "Effect": "Deny",
      "Action": [
        "ec2:RunInstances",
        "rds:CreateDBInstance",
        "s3:CreateBucket"
      ],
      "Resource": "*",
      "Condition": {
        "Null": {
          "aws:RequestTag/Environment": "true",
          "aws:RequestTag/Team": "true"
        }
      }
    }
  ]
}
```

### Key Takeaway

Use `default_tags` in the provider block to ensure every resource is tagged automatically. Enforce required tags with SCPs or AWS Config rules — do not rely on code review alone.

---

## 12. Over-Engineering Abstractions

Building layers of wrapper modules, custom providers, or internal platforms that abstract away the cloud provider to the point where engineers cannot understand, debug, or extend the infrastructure. The abstraction becomes harder to maintain than the underlying resources it was meant to simplify.

### Why It Happens

- Platform teams want to protect application teams from cloud complexity
- "What if we switch cloud providers?" — building for a migration that never happens
- Desire for a "single interface" that hides all provider-specific details
- Resume-driven development — over-engineering to showcase advanced Terraform skills

### What Goes Wrong

```
Over-abstraction layers:

Application Team
       │
       ▼
┌──────────────────┐
│  Internal Module  │  "deploy_service" — 200-line wrapper
│  Wrapper v3.2     │  that calls another internal module
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Platform Module  │  "cloud_resource" — 500-line abstraction
│  Wrapper v2.1     │  that maps generic names to AWS resources
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  AWS Provider     │  The actual Terraform resource
│  Resource         │  (3 lines of HCL)
└──────────────────┘

Engineer needs to add a health check timeout:
→ Internal Module doesn't expose the variable
→ Platform Module doesn't expose the variable
→ 3 PRs across 3 repos to add one parameter
→ 2 weeks to ship a 1-line change
```

### The Fix

❌ **Bad — unnecessary cloud-agnostic abstraction:**

```hcl
# modules/cloud-agnostic-database/main.tf — 400 lines
variable "cloud_provider" {
  type = string  # "aws", "azure", "gcp"
}

resource "aws_db_instance" "main" {
  count          = var.cloud_provider == "aws" ? 1 : 0
  engine         = var.engine
  instance_class = var.aws_instance_class
  # ...
}

resource "azurerm_mssql_server" "main" {
  count   = var.cloud_provider == "azure" ? 1 : 0
  version = "12.0"
  # ...
}

resource "google_sql_database_instance" "main" {
  count            = var.cloud_provider == "gcp" ? 1 : 0
  database_version = var.gcp_database_version
  # ...
}

# Result: nobody understands the module, it's always behind
# the native provider features, and the cloud migration never happens.
```

✅ **Good — thin, provider-native modules with clear contracts:**

```hcl
# modules/rds-postgres/main.tf — one thing, one provider, simple
resource "aws_db_instance" "main" {
  identifier     = "${var.environment}-${var.name}"
  engine         = "postgres"
  engine_version = var.engine_version
  instance_class = var.instance_class
  multi_az       = var.multi_az

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]

  tags = var.common_tags
}

resource "aws_db_subnet_group" "main" {
  name       = "${var.environment}-${var.name}"
  subnet_ids = var.subnet_ids
}

resource "aws_security_group" "db" {
  name   = "${var.environment}-${var.name}-db"
  vpc_id = var.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = var.allowed_security_group_ids
  }
}
```

```hcl
# modules/rds-postgres/variables.tf — fewer than 15 variables
variable "name" {
  description = "Database name"
  type        = string
}

variable "environment" {
  description = "Environment (dev, staging, prod)"
  type        = string
}

variable "instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.r5.large"
}

variable "engine_version" {
  description = "PostgreSQL version"
  type        = string
  default     = "15.4"
}

variable "multi_az" {
  description = "Enable Multi-AZ deployment"
  type        = bool
  default     = false
}

variable "subnet_ids" {
  description = "Subnet IDs for the DB subnet group"
  type        = list(string)
}

variable "vpc_id" {
  description = "VPC ID for security group"
  type        = string
}

variable "allowed_security_group_ids" {
  description = "Security groups allowed to connect"
  type        = list(string)
}

variable "common_tags" {
  description = "Tags applied to all resources"
  type        = map(string)
  default     = {}
}
```

### Key Takeaway

Abstractions should reduce complexity, not add layers. Build modules that wrap a single provider's resources, expose the parameters people actually need, and stay close enough to the provider documentation that engineers can look up any attribute themselves.

---

## Quick Reference Checklist

### Code and Version Control

- [ ] All infrastructure changes are made through code, never via the console
- [ ] Terraform and provider versions are pinned in `required_providers` and `.terraform.lock.hcl`
- [ ] No secrets, passwords, or API keys exist in `.tf` files, `.tfvars` files, or git history
- [ ] Infrastructure code is reviewed via pull requests before merging
- [ ] `.gitignore` excludes `.terraform/`, `*.tfstate`, `*.tfstate.backup`, and `*.tfplan`

### State Management

- [ ] State is stored in a remote backend (S3 + DynamoDB, Terraform Cloud, etc.)
- [ ] State backend encryption is enabled (SSE-KMS or equivalent)
- [ ] State locking is enabled to prevent concurrent modifications
- [ ] State files are split by domain — networking, data stores, and services have separate state
- [ ] No single state file contains more than 200 resources

### Modules and Reusability

- [ ] Common patterns (VPC, database, service) are extracted into reusable modules
- [ ] Modules have fewer than 15 input variables and clear documentation
- [ ] Modules are versioned and consumed via version-pinned `source` references
- [ ] No copy-pasted infrastructure blocks exist across environments
- [ ] Modules do not use boolean flags to conditionally create unrelated resources

### CI/CD and Safety

- [ ] `terraform plan` runs automatically on every pull request
- [ ] Plan output is posted to the PR for team review before apply
- [ ] `terraform apply` runs only in CI/CD, never from developer laptops
- [ ] Production apply requires manual approval via environment protection rules
- [ ] Critical resources use `lifecycle { prevent_destroy = true }`

### Drift and Consistency

- [ ] Drift detection runs on a schedule (at least weekly)
- [ ] Drift alerts are sent to the owning team via Slack, email, or GitHub Issues
- [ ] `lifecycle { ignore_changes }` is used only for values legitimately managed outside Terraform
- [ ] Console-created resources are imported or deleted — never left as unmanaged drift

### Operations and Tagging

- [ ] All resources have `Environment`, `Team`, `Project`, and `ManagedBy` tags
- [ ] `default_tags` in the provider block ensures consistent tagging automatically
- [ ] SCPs or AWS Config rules enforce required tags at the account level
- [ ] Cost reports can attribute spend to teams and environments using tags
- [ ] Tagging standards are documented and enforced in code review

---

## Next Steps

1. **Score yourself** — Use the [Quick Reference Checklist](#quick-reference-checklist) and score your current infrastructure code. Any unchecked item is a potential anti-pattern.
2. **Fix critical severity first** — Address 🔴 Critical anti-patterns (#1–#5) before high and medium ones.
3. **Read the companion guide** — [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) describes the correct patterns to replace these anti-patterns.
4. **Understand state management** — [04-STATE-MANAGEMENT.md](04-STATE-MANAGEMENT.md) covers remote backends, locking, and state file organisation in depth.
5. **Learn modules and reuse** — [05-MODULES-AND-REUSE.md](05-MODULES-AND-REUSE.md) covers how to write, version, and publish reusable Terraform modules.
6. **Secure your secrets** — [07-SECRETS-AND-VARIABLES.md](07-SECRETS-AND-VARIABLES.md) covers secret injection, encryption, and variable management.

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026 | Initial IaC Anti-Patterns documentation |
