# Infrastructure as Code Fundamentals

## Table of Contents

1. [Overview](#overview)
2. [What is Infrastructure as Code](#what-is-infrastructure-as-code)
3. [Declarative vs Imperative Approaches](#declarative-vs-imperative-approaches)
4. [IaC Tool Landscape](#iac-tool-landscape)
5. [The IaC Lifecycle](#the-iac-lifecycle)
6. [State Management Fundamentals](#state-management-fundamentals)
7. [Idempotency and Convergence](#idempotency-and-convergence)
8. [Drift Detection](#drift-detection)
9. [The IaC Workflow in Teams](#the-iac-workflow-in-teams)
10. [IaC and CI/CD Integration](#iac-and-cicd-integration)
11. [Prerequisites](#prerequisites)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

This documentation provides a comprehensive introduction to Infrastructure as Code (IaC). It covers the foundational concepts, declarative vs imperative approaches, the IaC tool landscape, lifecycle management, state management, idempotency, drift detection, team workflows, and CI/CD integration patterns needed to manage infrastructure reliably and at scale.

### Target Audience

- **Developers** building applications that depend on cloud infrastructure and contributing to IaC codebases alongside application code
- **DevOps Engineers** designing, building, and maintaining IaC configurations and automation pipelines for provisioning and managing infrastructure
- **Site Reliability Engineers (SREs)** ensuring infrastructure reliability, drift remediation, and operational consistency across environments
- **Platform Engineers** creating reusable modules, self-service infrastructure platforms, and golden paths for development teams
- **Architects** evaluating IaC strategies, tool selection, multi-cloud patterns, and organizational standards for infrastructure management

### Scope

- What Infrastructure as Code is and why it replaces manual provisioning
- Declarative vs imperative approaches and when to use each
- The IaC tool landscape: Terraform, OpenTofu, Pulumi, CloudFormation, Bicep, CDK, and Crossplane
- The IaC lifecycle: Write, Plan, Apply, and Destroy
- State management fundamentals: local vs remote state, locking, and state backends
- Idempotency and convergence: why IaC operations must be repeatable and self-correcting
- Drift detection: identifying and remediating configuration drift
- Team workflows: PR-based infrastructure changes, plan-on-PR, apply-on-merge, approval gates
- CI/CD integration patterns for IaC pipelines

---

## What is Infrastructure as Code

**Infrastructure as Code (IaC)** is the practice of managing and provisioning computing infrastructure through machine-readable configuration files rather than manual processes or interactive configuration tools. Instead of clicking through cloud consoles or running ad-hoc scripts, teams define their infrastructure in version-controlled code that can be reviewed, tested, and applied consistently.

### The Problem with Manual Provisioning

Before IaC, teams provisioned infrastructure manually — clicking through cloud consoles, running one-off commands, and documenting procedures in wikis that quickly became outdated:

```
  Manual Infrastructure Provisioning — What Could Go Wrong?

  ┌─────────────────────────────────────────────────────────────┐
  │                    The Manual Process                        │
  │                                                             │
  │  1. Developer requests infrastructure via ticket            │
  │  2. Ops engineer reads a wiki runbook                       │
  │  3. Ops clicks through the cloud console                    │
  │  4. Ops runs ad-hoc CLI commands from memory                │
  │  5. Ops updates the wiki (maybe)                            │
  │  6. Nobody knows exactly what was configured                │
  │  7. Months later, "Who created this? Why does it exist?"    │
  │                                                             │
  │  Result:                                                    │
  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐    │
  │  │ Snowflake  │  │ Environment│  │  No audit trail    │    │
  │  │ servers    │  │ drift      │  │  "Works on my      │    │
  │  │ (unique,   │  │ (dev ≠ stg │  │   cloud account"   │    │
  │  │ fragile)   │  │  ≠ prod)   │  │                    │    │
  │  └────────────┘  └────────────┘  └────────────────────┘    │
  └─────────────────────────────────────────────────────────────┘
```

**Common problems with manual provisioning:**

- **Snowflake environments** — every server is hand-crafted and unique, making them impossible to reproduce
- **Configuration drift** — environments diverge over time as ad-hoc changes accumulate
- **No audit trail** — who changed what, when, and why is lost to Slack messages and tribal knowledge
- **Slow provisioning** — requesting infrastructure takes days or weeks through ticketing systems
- **Error-prone** — humans make mistakes when clicking through 47 configuration screens
- **No review process** — infrastructure changes bypass code review, testing, and approval workflows
- **Difficult disaster recovery** — recreating an environment from memory is slow and unreliable

### How IaC Solves These Problems

IaC treats infrastructure configuration the same way software engineering treats application code:

```
  Infrastructure as Code — The Paradigm Shift

  Manual Provisioning                    Infrastructure as Code
  ─────────────────────                  ──────────────────────
  Cloud console clicks          ───►     Configuration files (HCL, YAML, TypeScript)
  Ad-hoc CLI commands           ───►     Version-controlled repositories
  Wiki runbooks                 ───►     Self-documenting code
  Tribal knowledge              ───►     Code review and pull requests
  "It works on my account"      ───►     Identical environments from the same code
  Hours to provision            ───►     Minutes to provision
  Manual change tracking        ───►     Git history and audit logs
  Hope-based disaster recovery  ───►     Recreate everything from code
```

### Core Benefits of IaC

| Benefit | Description |
|---|---|
| **Reproducibility** | The same code produces the same infrastructure every time, in any account or region |
| **Version control** | Infrastructure changes go through Git — full history, blame, revert, and branching |
| **Code review** | Pull requests for infrastructure changes, just like application code |
| **Automation** | Provisioning is automated end-to-end; no manual steps, no console clicking |
| **Self-documentation** | The code is the documentation — always accurate, always up to date |
| **Testing** | Infrastructure can be validated, linted, and tested before applying changes |
| **Disaster recovery** | Recreate entire environments from code in minutes, not days |
| **Collaboration** | Teams work on infrastructure concurrently using branches and pull requests |
| **Compliance** | Policy as code enforces organizational standards automatically |
| **Cost visibility** | Plan output shows exactly what resources will be created, changed, or destroyed |

### IaC is Not Just Automation

A common misconception is that IaC is simply "scripting infrastructure." There is an important distinction:

```
  Automation Scripts vs Infrastructure as Code

  Automation (Imperative Scripts)          Infrastructure as Code (Declarative)
  ──────────────────────────────           ────────────────────────────────────
  "Run these 50 commands in order"         "Here is the desired end state"
  Breaks if run twice                      Safe to run repeatedly (idempotent)
  No awareness of current state            Compares desired state to actual state
  Must handle every edge case              The tool handles ordering and dependencies
  Difficult to reason about                Easy to read and review
  Example: Bash script with                Example: Terraform HCL, CloudFormation
    aws cli commands                         YAML, Pulumi TypeScript
```

---

## Declarative vs Imperative Approaches

The two fundamental approaches to IaC differ in **what you specify** — the desired end state (declarative) or the exact steps to get there (imperative). Understanding this distinction is critical for choosing the right tool and writing effective IaC.

### Declarative: Describe the Desired State

In a declarative approach, you define **what** the infrastructure should look like, and the IaC tool figures out **how** to make it happen. The tool compares the desired state to the current state and computes the necessary changes.

```hcl
# Declarative example: Terraform HCL
# "I want a VPC with these properties. Make it so."

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name        = "production-vpc"
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-${count.index + 1}"
    Tier = "public"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "production-igw"
  }
}
```

**Key characteristics of declarative IaC:**

- You describe the end state, not the steps
- The tool computes a plan (diff) between current and desired state
- Running the same configuration again produces no changes (idempotent)
- Dependency ordering is handled automatically by the tool
- Easier to understand at a glance — "this is what exists"

### Imperative: Describe the Steps

In an imperative approach, you define the exact sequence of operations to execute. You write code that calls APIs directly, and you control the ordering, error handling, and state management.

```typescript
// Imperative example: Pulumi TypeScript
// "Execute these steps to create the infrastructure."

import * as aws from "@pulumi/aws";

const vpc = new aws.ec2.Vpc("main", {
    cidrBlock: "10.0.0.0/16",
    enableDnsSupport: true,
    enableDnsHostnames: true,
    tags: {
        Name: "production-vpc",
        Environment: "production",
        ManagedBy: "pulumi",
    },
});

const azs = aws.getAvailabilityZones({ state: "available" });

const publicSubnets: aws.ec2.Subnet[] = [];
for (let i = 0; i < 3; i++) {
    const subnet = new aws.ec2.Subnet(`public-subnet-${i + 1}`, {
        vpcId: vpc.id,
        cidrBlock: `10.0.${i}.0/24`,
        availabilityZone: azs.then(az => az.names[i]),
        mapPublicIpOnLaunch: true,
        tags: {
            Name: `public-subnet-${i + 1}`,
            Tier: "public",
        },
    });
    publicSubnets.push(subnet);
}

const igw = new aws.ec2.InternetGateway("main", {
    vpcId: vpc.id,
    tags: {
        Name: "production-igw",
    },
});
```

**Key characteristics of imperative IaC:**

- You describe the steps, using general-purpose programming constructs
- Full access to loops, conditionals, functions, classes, and libraries
- You can use the same language as your application code (TypeScript, Python, Go, C#)
- More flexibility for complex logic, dynamic configuration, and abstractions
- Requires more discipline to maintain idempotency

### Declarative vs Imperative Comparison

| Aspect | Declarative | Imperative |
|---|---|---|
| **You specify** | Desired end state | Exact steps to execute |
| **Dependency ordering** | Handled by the tool automatically | You control the ordering |
| **Idempotency** | Built-in — the tool computes diffs | Must be implemented by the author |
| **Language** | Domain-specific (HCL, YAML, Bicep) | General-purpose (TypeScript, Python, Go) |
| **Learning curve** | Lower — learn the DSL | Higher — learn the SDK + language |
| **Flexibility** | Limited by DSL constructs | Unlimited — full programming language |
| **Refactoring** | Limited — no native functions or classes | Full IDE support — refactoring, types, tests |
| **Testing** | Plan-based validation, policy checks | Unit tests, integration tests, type checking |
| **Ecosystem** | HCL modules, Terraform Registry | npm, PyPI, NuGet — full package ecosystems |
| **Best for** | Standard infrastructure patterns | Complex, dynamic, or highly customized setups |
| **Tools** | Terraform, OpenTofu, CloudFormation, Bicep | Pulumi, AWS CDK, Crossplane (YAML but reconciles) |

### When to Choose Each Approach

```
  Decision Guide: Declarative vs Imperative

  Choose Declarative when:                Choose Imperative when:
  ─────────────────────────               ──────────────────────────
  ✓ Standard cloud resources              ✓ Complex conditional logic
  ✓ Team has mixed experience levels      ✓ Team is fluent in TypeScript/Python/Go
  ✓ Simple, well-understood patterns      ✓ Dynamic resource generation
  ✓ You want the tool to handle ordering  ✓ You need full testing frameworks
  ✓ Wide community module ecosystem       ✓ Tight integration with app code
  ✓ Compliance requires readable configs  ✓ Building reusable abstractions (classes)
```

---

## IaC Tool Landscape

The IaC ecosystem includes multiple tools, each with different design philosophies, language choices, and cloud provider support. Understanding the landscape helps teams choose the right tool for their needs.

### Tool Overview

```
  IaC Tool Landscape

  Declarative DSL                 General-Purpose Languages
  ───────────────                 ─────────────────────────
  ┌─────────────┐                ┌─────────────┐
  │  Terraform  │                │   Pulumi    │
  │  (HCL)      │                │  (TS, Py,   │
  │  Multi-cloud│                │   Go, C#)   │
  └──────┬──────┘                └──────┬──────┘
         │                              │
  ┌──────┴──────┐                ┌──────┴──────┐
  │  OpenTofu   │                │   AWS CDK   │
  │  (HCL)      │                │  (TS, Py,   │
  │  OSS Fork   │                │   Java, C#) │
  └─────────────┘                └─────────────┘

  Cloud-Native DSL                Kubernetes-Native
  ────────────────                ─────────────────
  ┌─────────────┐                ┌─────────────┐
  │ CloudFormat.│                │ Crossplane  │
  │ (YAML/JSON) │                │ (YAML CRDs) │
  │ AWS-only    │                │ K8s-native  │
  └─────────────┘                └─────────────┘
  ┌─────────────┐
  │   Bicep     │
  │ (Bicep DSL) │
  │ Azure-only  │
  └─────────────┘
```

### Feature Comparison

| Feature | Terraform | OpenTofu | Pulumi | CloudFormation | Bicep | AWS CDK | Crossplane |
|---|---|---|---|---|---|---|---|
| **Language** | HCL | HCL | TS, Py, Go, C#, Java | YAML / JSON | Bicep DSL | TS, Py, Java, C#, Go | YAML (CRDs) |
| **Cloud support** | Multi-cloud | Multi-cloud | Multi-cloud | AWS only | Azure only | AWS only | Multi-cloud |
| **State management** | Built-in (local/remote) | Built-in (local/remote) | Built-in (Pulumi Cloud/self-hosted) | Managed by AWS | Managed by Azure | Managed by AWS | Kubernetes etcd |
| **Plan / Preview** | `terraform plan` | `tofu plan` | `pulumi preview` | Change Sets | What-If | `cdk diff` | Dry-run via controllers |
| **Module / Reuse** | Modules (Registry) | Modules (Registry) | Component Resources, classes | Nested Stacks, Modules | Modules | Constructs (L1, L2, L3) | Compositions, XRDs |
| **License** | BSL 1.1 | MPL 2.0 (OSS) | Apache 2.0 | Proprietary (AWS) | MIT | Apache 2.0 | Apache 2.0 |
| **Policy as code** | Sentinel, OPA | OPA, Checkov | CrossGuard, OPA | AWS Config Rules | Azure Policy | AWS Config Rules | OPA Gatekeeper |
| **Maturity** | Very high | Growing | High | Very high | High | High | Growing |
| **Learning curve** | Low-medium | Low-medium | Medium-high | Medium | Low | Medium-high | High |
| **Best for** | Multi-cloud, standard patterns | Multi-cloud, OSS-first teams | Dev teams, complex logic | AWS-native shops | Azure-native shops | AWS devs who prefer code | K8s-native platforms |

### Terraform

**Terraform** is the most widely adopted IaC tool. It uses HashiCorp Configuration Language (HCL), a declarative DSL designed specifically for infrastructure configuration. Terraform supports hundreds of providers for cloud platforms, SaaS services, and internal APIs.

```hcl
# Terraform: Create an S3 bucket with versioning
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "data" {
  bucket = "my-app-data-bucket"

  tags = {
    Environment = "production"
    Team        = "platform"
  }
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

### OpenTofu

**OpenTofu** is an open-source fork of Terraform, created after HashiCorp changed Terraform's license from MPL 2.0 to BSL 1.1. OpenTofu is maintained by the Linux Foundation and is fully compatible with Terraform configurations and providers.

### Pulumi

**Pulumi** lets you write IaC using general-purpose programming languages. Instead of learning a DSL, you use TypeScript, Python, Go, C#, or Java with full IDE support, type checking, and testing frameworks.

```typescript
// Pulumi: Create an S3 bucket with versioning (TypeScript)
import * as aws from "@pulumi/aws";

const bucket = new aws.s3.BucketV2("data", {
    bucket: "my-app-data-bucket",
    tags: {
        Environment: "production",
        Team: "platform",
    },
});

const versioning = new aws.s3.BucketVersioningV2("data", {
    bucket: bucket.id,
    versioningConfiguration: {
        status: "Enabled",
    },
});

export const bucketName = bucket.bucket;
```

### CloudFormation

**AWS CloudFormation** is AWS's native IaC service. Templates are written in YAML or JSON and describe AWS resources declaratively. CloudFormation manages state internally — there is no external state file to manage.

```yaml
# CloudFormation: Create an S3 bucket with versioning
AWSTemplateFormatVersion: '2010-09-09'
Description: S3 bucket with versioning enabled

Resources:
  DataBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-app-data-bucket
      VersioningConfiguration:
        Status: Enabled
      Tags:
        - Key: Environment
          Value: production
        - Key: Team
          Value: platform

Outputs:
  BucketName:
    Description: Name of the S3 bucket
    Value: !Ref DataBucket
  BucketArn:
    Description: ARN of the S3 bucket
    Value: !GetAtt DataBucket.Arn
```

### Bicep

**Bicep** is Azure's domain-specific language for deploying Azure resources. It compiles down to ARM (Azure Resource Manager) templates but provides a cleaner, more concise syntax.

### AWS CDK

**AWS CDK** (Cloud Development Kit) lets you define AWS infrastructure using imperative code in TypeScript, Python, Java, C#, or Go. CDK synthesizes your code into CloudFormation templates.

### Crossplane

**Crossplane** is a Kubernetes-native IaC tool that extends the Kubernetes API to provision and manage external cloud resources using Custom Resource Definitions (CRDs) and controllers.

---

## The IaC Lifecycle

Every IaC tool follows a similar lifecycle: **Write** the configuration, **Plan** the changes, **Apply** the changes to real infrastructure, and optionally **Destroy** the infrastructure when it is no longer needed.

```
  The IaC Lifecycle

  ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐
  │          │      │          │      │          │      │          │
  │  Write   │─────►│   Plan   │─────►│  Apply   │─────►│ Destroy  │
  │          │      │          │      │          │      │          │
  └──────────┘      └──────────┘      └──────────┘      └──────────┘
       │                 │                 │                  │
       ▼                 ▼                 ▼                  ▼
  Define infra      Preview changes   Execute changes    Tear down all
  in code (HCL,     (what will be     against cloud      resources when
  YAML, TS, Py)     created, changed  provider APIs      no longer needed
                    or destroyed)
```

### Phase 1: Write

The Write phase is where you define infrastructure in configuration files. This includes choosing resources, setting properties, defining dependencies, and organizing code into modules.

```hcl
# Write phase: Define a web application infrastructure
# main.tf

module "networking" {
  source = "./modules/networking"

  vpc_cidr         = "10.0.0.0/16"
  environment      = var.environment
  availability_zones = var.availability_zones
}

module "database" {
  source = "./modules/database"

  vpc_id          = module.networking.vpc_id
  subnet_ids      = module.networking.private_subnet_ids
  instance_class  = var.db_instance_class
  engine_version  = "15.4"
  environment     = var.environment
}

module "application" {
  source = "./modules/application"

  vpc_id          = module.networking.vpc_id
  subnet_ids      = module.networking.public_subnet_ids
  db_endpoint     = module.database.endpoint
  instance_count  = var.app_instance_count
  environment     = var.environment
}
```

### Phase 2: Plan

The Plan phase generates a preview of what changes will be made. The tool compares the desired state (your configuration) to the current state (what actually exists in the cloud) and computes a diff.

```bash
# Plan phase: Preview changes before applying
$ terraform plan -out=tfplan

Terraform will perform the following actions:

  # module.networking.aws_vpc.main will be created
  + resource "aws_vpc" "main" {
      + arn                  = (known after apply)
      + cidr_block           = "10.0.0.0/16"
      + enable_dns_hostnames = true
      + enable_dns_support   = true
      + id                   = (known after apply)
      + tags                 = {
          + "Environment" = "production"
          + "Name"        = "production-vpc"
        }
    }

  # module.database.aws_db_instance.main will be created
  + resource "aws_db_instance" "main" {
      + allocated_storage    = 100
      + engine               = "postgres"
      + engine_version       = "15.4"
      + instance_class       = "db.r6g.large"
      + id                   = (known after apply)
    }

  # module.application.aws_instance.web[0] will be created
  + resource "aws_instance" "web" {
      + ami                  = "ami-0abcdef1234567890"
      + instance_type        = "t3.medium"
      + id                   = (known after apply)
    }

Plan: 12 to add, 0 to change, 0 to destroy.
```

### Phase 3: Apply

The Apply phase executes the planned changes against the cloud provider. The tool calls cloud APIs to create, update, or delete resources, then updates the state file to reflect the new reality.

```bash
# Apply phase: Execute the plan
$ terraform apply tfplan

module.networking.aws_vpc.main: Creating...
module.networking.aws_vpc.main: Creation complete after 3s [id=vpc-0abc123def456]
module.networking.aws_subnet.public[0]: Creating...
module.networking.aws_subnet.public[1]: Creating...
module.networking.aws_subnet.public[2]: Creating...
module.networking.aws_subnet.public[0]: Creation complete after 2s
module.networking.aws_subnet.public[1]: Creation complete after 2s
module.networking.aws_subnet.public[2]: Creation complete after 2s
module.database.aws_db_instance.main: Creating...
module.database.aws_db_instance.main: Still creating... [2m elapsed]
module.database.aws_db_instance.main: Creation complete after 4m12s
module.application.aws_instance.web[0]: Creating...
module.application.aws_instance.web[0]: Creation complete after 30s

Apply complete! Resources: 12 added, 0 changed, 0 destroyed.

Outputs:

app_url     = "http://app-lb-123456.us-east-1.elb.amazonaws.com"
db_endpoint = "prod-db.cluster-abc123.us-east-1.rds.amazonaws.com"
```

### Phase 4: Destroy

The Destroy phase tears down all resources defined in the configuration. This is essential for ephemeral environments (dev, test, demo) and cost management.

```bash
# Destroy phase: Tear down all resources
$ terraform destroy

Terraform will perform the following actions:

  # module.application.aws_instance.web[0] will be destroyed
  - resource "aws_instance" "web" {
      - ami           = "ami-0abcdef1234567890" -> null
      - instance_type = "t3.medium" -> null
      - id            = "i-0abc123def456" -> null
    }

  # module.database.aws_db_instance.main will be destroyed
  - resource "aws_db_instance" "main" {
      - engine         = "postgres" -> null
      - instance_class = "db.r6g.large" -> null
      - id             = "prod-db" -> null
    }

  # module.networking.aws_vpc.main will be destroyed
  - resource "aws_vpc" "main" {
      - cidr_block = "10.0.0.0/16" -> null
      - id         = "vpc-0abc123def456" -> null
    }

Plan: 0 to add, 0 to change, 12 to destroy.

Do you really want to destroy all resources? (yes/no): yes

Destroy complete! Resources: 12 destroyed.
```

---

## State Management Fundamentals

**State** is the mechanism IaC tools use to track the mapping between your configuration and the real-world resources that exist in the cloud. State is the single most important concept to understand in IaC — mismanaging state is the source of the most common and most dangerous IaC failures.

### What is State?

State is a record of every resource your IaC configuration manages. It stores:

- The resource type and name in your configuration
- The resource ID in the cloud provider (e.g., `vpc-0abc123def456`)
- All current attribute values (CIDR block, tags, status, ARN)
- Dependencies between resources
- Metadata about the last apply operation

```
  State: The Bridge Between Code and Reality

  Configuration (Code)          State File              Cloud Provider
  ────────────────────          ──────────              ──────────────
  resource "aws_vpc" "main" {   {                       VPC: vpc-0abc123
    cidr_block = "10.0.0.0/16"    "aws_vpc.main": {      CIDR: 10.0.0.0/16
    tags = {                        "id": "vpc-0abc123",  DNS: enabled
      Name = "production-vpc"       "cidr_block":         Tags:
    }                                 "10.0.0.0/16",        Name=production-vpc
  }                                 "tags": {
                                      "Name":
                                        "production-vpc"
                                    }
                                  }
                                }

  Desired State          ◄───►  Known State       ◄───►  Actual State
  (what you want)               (what Terraform         (what really exists
                                 last saw)               in the cloud)
```

### Why State Matters

Without state, the IaC tool would have no way to:

| Capability | Why State is Required |
|---|---|
| **Compute diffs** | Compare desired state to current state and determine what needs to change |
| **Track resource IDs** | Know which cloud resource corresponds to which block in your code |
| **Detect drift** | Compare the state file to the actual cloud state and find discrepancies |
| **Handle dependencies** | Know the order in which resources must be created or destroyed |
| **Support refactoring** | Move or rename resources in code without destroying and recreating them |
| **Enable collaboration** | Multiple team members work on the same infrastructure |

### Local vs Remote State

By default, most IaC tools store state locally in a file on disk. This works for experimentation but fails in any team or production scenario.

```
  Local State vs Remote State

  Local State                              Remote State
  ───────────                              ────────────
  ┌──────────────────┐                     ┌──────────────────────────┐
  │ terraform.tfstate│                     │  Remote Backend          │
  │ (on your laptop) │                     │  (S3, GCS, Azure Blob,  │
  └────────┬─────────┘                     │   Terraform Cloud)       │
           │                               └────────────┬─────────────┘
           ▼                                            ▼
  Problems:                                Benefits:
  • Only you can run apply                 • Shared across the team
  • Lost if laptop dies                    • Durable and backed up
  • No locking — concurrent                • Locking prevents conflicts
    applies corrupt state                  • Encrypted at rest
  • Not encrypted                          • Audit trail of changes
  • No audit trail                         • Required for CI/CD pipelines
```

**Configuring a remote backend (Terraform example):**

```hcl
# Remote state configuration — S3 backend with DynamoDB locking
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "production/networking/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

```hcl
# The DynamoDB table used for state locking
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name    = "terraform-state-lock"
    Purpose = "Terraform state locking"
  }
}
```

### State Locking

State locking prevents two people (or two CI/CD pipeline runs) from modifying the same state file simultaneously. Without locking, concurrent applies can corrupt state and leave infrastructure in an inconsistent state.

```
  State Locking — Preventing Concurrent Modifications

  Without Locking:                         With Locking:
  ────────────────                         ─────────────
  Engineer A: terraform apply              Engineer A: terraform apply
  Engineer B: terraform apply              Engineer B: terraform apply
       │              │                         │              │
       ▼              ▼                         ▼              ▼
  Both read state simultaneously           A acquires lock ✓
  Both compute plans                       B blocked: "state locked" ✗
  Both write state                         A applies and releases lock
  STATE CORRUPTED ✗                        B acquires lock, applies ✓
```

| Backend | Locking Mechanism |
|---|---|
| **S3** | DynamoDB table with `LockID` hash key |
| **GCS** | Built-in object locking |
| **Azure Blob** | Blob lease mechanism |
| **Terraform Cloud** | Built-in — automatic locking per workspace |
| **Consul** | Key/value store with session-based locks |
| **PostgreSQL** | Advisory locks |

---

## Idempotency and Convergence

### Why IaC Must Be Idempotent

**Idempotency** means that applying the same configuration multiple times produces the same result as applying it once. This is a fundamental property of reliable IaC — if an apply fails halfway through, you should be able to re-run it safely without creating duplicates or corrupting existing resources.

```
  Idempotency in Practice

  First apply:                Second apply (no changes):
  ────────────                ──────────────────────────
  $ terraform apply           $ terraform apply

  + aws_vpc.main: Creating    aws_vpc.main: Refreshing state...
  + aws_vpc.main: Created     aws_subnet.public[0]: Refreshing...
  + aws_subnet.public[0]:     aws_subnet.public[1]: Refreshing...
    Creating
  + aws_subnet.public[0]:     No changes. Your infrastructure
    Created                   matches the configuration.

  Apply complete!             Apply complete!
  3 added, 0 changed,        0 added, 0 changed,
  0 destroyed.                0 destroyed.
```

**Why idempotency matters:**

- **Safe retries** — if an apply fails partway through (network timeout, API rate limit), re-running it will pick up where it left off
- **CI/CD safety** — pipelines can run apply on every merge without fear of creating duplicate resources
- **Drift remediation** — running apply after manual changes brings infrastructure back to the desired state
- **Disaster recovery** — applying the same configuration to a new account recreates the infrastructure identically

### How Declarative Tools Achieve Idempotency

Declarative tools (Terraform, CloudFormation, Bicep) achieve idempotency through the **state-based diff** approach:

1. Read the current state (from state file and cloud APIs)
2. Compare current state to desired state (your configuration)
3. Compute the minimal set of changes needed
4. Apply only those changes

```
  State-Based Diff — The Idempotency Engine

  Configuration          State File          Cloud Provider
  (desired state)        (known state)       (actual state)
       │                      │                    │
       └──────────┬───────────┘                    │
                  │                                │
                  ▼                                │
          ┌──────────────┐                         │
          │  Diff Engine │◄────────────────────────┘
          │              │    Refresh: read actual
          │  Desired vs  │    state from cloud APIs
          │  Actual      │
          └──────┬───────┘
                 │
                 ▼
          ┌──────────────┐
          │  Change Set  │
          │              │
          │  + create 2  │
          │  ~ update 1  │
          │  - delete 0  │
          └──────────────┘
```

### Convergence

**Convergence** is the property that repeated application of a configuration will bring the system closer to the desired state until it fully matches. Some tools (particularly Kubernetes-based tools like Crossplane) use a **reconciliation loop** — a controller continuously watches for drift and corrects it automatically.

```
  Convergence Loop (Kubernetes-Native Tools)

  ┌─────────────────────────────────────────────────┐
  │                                                 │
  │   ┌──────────┐    ┌──────────┐    ┌──────────┐ │
  │   │ Observe  │───►│ Compare  │───►│  Act     │ │
  │   │ actual   │    │ desired  │    │ (create, │ │
  │   │ state    │    │ vs actual│    │  update, │ │
  │   └──────────┘    └──────────┘    │  delete) │ │
  │        ▲                          └────┬─────┘ │
  │        │                               │       │
  │        └───────────────────────────────┘       │
  │              Continuous reconciliation          │
  └─────────────────────────────────────────────────┘

  The loop runs every 30-300 seconds (configurable).
  If someone manually changes a resource, the controller
  detects the drift and corrects it automatically.
```

---

## Drift Detection

### What is Drift?

**Drift** occurs when the actual state of infrastructure diverges from the state declared in your IaC configuration. Drift is one of the most dangerous problems in infrastructure management because it means your code no longer accurately represents reality.

```
  Configuration Drift — How It Happens

  Day 1: Everything matches
  ─────────────────────────
  Code:   instance_type = "t3.medium"
  State:  instance_type = "t3.medium"
  Cloud:  instance_type = "t3.medium"    ✓ All consistent

  Day 30: Someone makes a manual change
  ──────────────────────────────────────
  Code:   instance_type = "t3.medium"
  State:  instance_type = "t3.medium"
  Cloud:  instance_type = "t3.xlarge"    ✗ DRIFT!
          (changed via console)

  Day 31: Next terraform apply
  ────────────────────────────
  Terraform detects the drift during refresh:
    ~ aws_instance.web
        instance_type: "t3.xlarge" → "t3.medium"

  Terraform will revert the manual change!
```

### Common Causes of Drift

| Cause | Description | Prevention |
|---|---|---|
| **Console changes** | Someone modifies a resource through the cloud console | Restrict console write access; use read-only roles |
| **CLI commands** | Ad-hoc `aws`, `az`, or `gcloud` commands modify resources | Enforce all changes through IaC pipelines |
| **Auto-scaling** | Cloud services automatically adjust resources (e.g., ASG instance count) | Ignore auto-managed attributes in IaC with lifecycle rules |
| **Automated patches** | Cloud provider applies security patches or version upgrades | Pin versions explicitly in configuration |
| **Other tools** | Another IaC configuration or script modifies the same resource | Define clear ownership boundaries between IaC projects |
| **Emergency fixes** | Hotfixes applied manually during an incident | Follow up with IaC code changes after the incident |

### How Tools Detect Drift

| Tool | Drift Detection Method | Command |
|---|---|---|
| **Terraform** | Refresh state, then plan to see diffs | `terraform plan -refresh-only` |
| **OpenTofu** | Same as Terraform | `tofu plan -refresh-only` |
| **Pulumi** | Refresh state from cloud provider | `pulumi refresh` |
| **CloudFormation** | Built-in drift detection | `aws cloudformation detect-stack-drift` |
| **Bicep / ARM** | What-If operation | `az deployment group what-if` |
| **Crossplane** | Continuous reconciliation loop | Automatic — controller detects and corrects |

### Drift Remediation Strategies

```
  Drift Detected — Now What?

  Option 1: Revert to Code (Most Common)
  ───────────────────────────────────────
  Run terraform apply to overwrite the manual change
  and bring the cloud resource back to the declared state.

  Option 2: Update the Code
  ─────────────────────────
  If the manual change was intentional (e.g., emergency fix),
  update the IaC code to match the new reality, then run
  terraform apply to reconcile.

  Option 3: Import the Change
  ───────────────────────────
  If a new resource was created manually, use terraform import
  to bring it under IaC management without recreating it.

  Option 4: Ignore the Drift
  ──────────────────────────
  For attributes managed by the cloud provider (e.g., auto-scaling
  group desired count), use lifecycle ignore_changes to tell
  the tool not to revert those attributes.
```

```hcl
# Ignoring drift for auto-managed attributes
resource "aws_autoscaling_group" "web" {
  name                = "web-asg"
  min_size            = 2
  max_size            = 10
  desired_capacity    = 2
  vpc_zone_identifier = var.subnet_ids

  lifecycle {
    ignore_changes = [
      desired_capacity,  # Auto-scaling adjusts this dynamically
    ]
  }
}
```

---

## The IaC Workflow in Teams

In a team environment, infrastructure changes follow a structured workflow similar to application code changes — branch, write, review, plan, approve, and apply.

### PR-Based Infrastructure Workflow

```
  PR-Based IaC Workflow

  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  Branch  │───►│  Write   │───►│  Commit  │───►│  Open PR │
  │          │    │  IaC     │    │  & Push  │    │          │
  └──────────┘    └──────────┘    └──────────┘    └─────┬────┘
                                                        │
                                                        ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                    PR Automation                              │
  │                                                              │
  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────┐ │
  │  │  Validate   │  │    Plan     │  │  Post Plan as        │ │
  │  │  (fmt,      │  │  (terraform │  │  PR Comment          │ │
  │  │   lint,     │  │   plan)     │  │  (show diff of       │ │
  │  │   validate) │  │             │  │   infra changes)     │ │
  │  └─────────────┘  └─────────────┘  └──────────────────────┘ │
  └──────────────────────────────────────────────────────────────┘
                                                        │
                                                        ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  Code    │───►│  Approve │───►│  Merge   │───►│  Apply   │
  │  Review  │    │  (plan   │    │  to main │    │  (auto   │
  │          │    │   ok?)   │    │          │    │   or     │
  └──────────┘    └──────────┘    └──────────┘    │  manual) │
                                                  └──────────┘
```

### Plan-on-PR, Apply-on-Merge

The most common team pattern is to **run the plan when a PR is opened** (so reviewers can see what will change) and **run the apply when the PR is merged** (so changes are applied only after approval).

```yaml
# GitHub Actions: Plan on PR, Apply on merge
name: Infrastructure
on:
  pull_request:
    branches: [main]
    paths: ['infrastructure/**']
  push:
    branches: [main]
    paths: ['infrastructure/**']

jobs:
  plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.0

      - name: Terraform Init
        run: terraform init
        working-directory: infrastructure/

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan
        working-directory: infrastructure/

      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            *Triggered by @${{ github.actor }}*`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

  apply:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.0

      - name: Terraform Init
        run: terraform init
        working-directory: infrastructure/

      - name: Terraform Apply
        run: terraform apply -auto-approve
        working-directory: infrastructure/
```

### Approval Gates

For production infrastructure, teams add manual approval gates between plan and apply:

| Gate Type | When to Use | Implementation |
|---|---|---|
| **PR review** | Every infrastructure change | Branch protection rules — require 1-2 approvals |
| **Plan review** | Complex or risky changes | Post plan output as PR comment; reviewer verifies |
| **Environment approval** | Production deployments | GitHub Environments with required reviewers |
| **Policy check** | Compliance requirements | OPA, Sentinel, or Checkov gates in the pipeline |
| **Cost estimate** | Budget-sensitive changes | Infracost integration that posts cost diff to PR |

---

## IaC and CI/CD Integration

IaC pipelines follow a similar structure to application CI/CD pipelines but with IaC-specific stages for validation, planning, policy checking, and applying infrastructure changes.

### Pipeline Architecture

```
  IaC CI/CD Pipeline Architecture

  ┌─────────────────────────────────────────────────────────────────┐
  │                     Validation Stage                            │
  │                                                                 │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
  │  │   Format     │  │    Lint      │  │     Validate         │  │
  │  │  terraform   │  │  tflint,     │  │   terraform          │  │
  │  │  fmt -check  │  │  checkov     │  │   validate           │  │
  │  └──────────────┘  └──────────────┘  └──────────────────────┘  │
  └──────────────────────────┬──────────────────────────────────────┘
                             │
  ┌──────────────────────────▼──────────────────────────────────────┐
  │                      Plan Stage                                 │
  │                                                                 │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
  │  │  terraform   │  │   Policy     │  │    Cost              │  │
  │  │  plan        │  │   Check      │  │    Estimate          │  │
  │  │  -out=plan   │  │  (OPA /      │  │   (Infracost)       │  │
  │  └──────────────┘  │   Sentinel)  │  └──────────────────────┘  │
  │                     └──────────────┘                            │
  └──────────────────────────┬──────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │ Manual Approval │
                    │  (production)   │
                    └────────┬────────┘
                             │
  ┌──────────────────────────▼──────────────────────────────────────┐
  │                     Apply Stage                                 │
  │                                                                 │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
  │  │  terraform   │  │   Smoke      │  │    Notify            │  │
  │  │  apply       │  │   Tests      │  │   (Slack, PR         │  │
  │  │  tfplan      │  │  (validate   │  │    comment)          │  │
  │  └──────────────┘  │   endpoints) │  └──────────────────────┘  │
  │                     └──────────────┘                            │
  └─────────────────────────────────────────────────────────────────┘
```

### Pipeline Patterns for IaC

| Pattern | Description | When to Use |
|---|---|---|
| **Plan-on-PR, Apply-on-merge** | Plan runs on PRs for review; apply runs on merge to main | Standard pattern for most teams |
| **Workspace per environment** | Separate Terraform workspaces for dev, staging, production | When environments share the same code but differ in variables |
| **Directory per environment** | Separate directories with their own state for each environment | When environments differ significantly in configuration |
| **Scheduled drift detection** | Cron job runs `terraform plan` and alerts on unexpected changes | Production environments with strict change control |
| **Destroy on PR close** | Destroy ephemeral environments when a feature branch PR is closed | Preview environments, cost optimization |
| **Multi-stage promotion** | Apply to dev → staging → production with gates between each | Risk-sensitive organizations with compliance requirements |

### Environment Promotion

```
  Environment Promotion Strategy

  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │   main branch                                            │
  │   ┌───────────────────────────────────────────────────┐  │
  │   │                                                   │  │
  │   │   ┌──────────┐    ┌──────────┐    ┌──────────┐   │  │
  │   │   │   Dev    │───►│ Staging  │───►│   Prod   │   │  │
  │   │   │          │    │          │    │          │   │  │
  │   │   │ auto-    │    │ auto-    │    │ manual   │   │  │
  │   │   │ apply    │    │ apply    │    │ approval │   │  │
  │   │   │          │    │ + smoke  │    │ + smoke  │   │  │
  │   │   │          │    │   tests  │    │   tests  │   │  │
  │   │   └──────────┘    └──────────┘    └──────────┘   │  │
  │   │                                                   │  │
  │   └───────────────────────────────────────────────────┘  │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

### Example: Multi-Environment Pipeline

```yaml
# GitHub Actions: Multi-environment IaC pipeline
name: Infrastructure Deployment
on:
  push:
    branches: [main]
    paths: ['infrastructure/**']

jobs:
  deploy-dev:
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init -backend-config=environments/dev/backend.hcl
        working-directory: infrastructure/

      - name: Terraform Apply
        run: terraform apply -auto-approve -var-file=environments/dev/terraform.tfvars
        working-directory: infrastructure/

  deploy-staging:
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init -backend-config=environments/staging/backend.hcl
        working-directory: infrastructure/

      - name: Terraform Apply
        run: terraform apply -auto-approve -var-file=environments/staging/terraform.tfvars
        working-directory: infrastructure/

      - name: Smoke Tests
        run: ./scripts/smoke-test.sh staging

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production    # Requires manual approval
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init -backend-config=environments/production/backend.hcl
        working-directory: infrastructure/

      - name: Terraform Apply
        run: terraform apply -auto-approve -var-file=environments/production/terraform.tfvars
        working-directory: infrastructure/

      - name: Smoke Tests
        run: ./scripts/smoke-test.sh production
```

---

## Prerequisites

### Required Knowledge

Before working through the Infrastructure as Code series, you should be familiar with:

| Topic | Why It Matters |
|---|---|
| **Cloud computing basics** | IaC provisions cloud resources; you need to understand VPCs, compute, storage, and IAM |
| **Linux command line** | IaC tools run from the terminal; debugging requires comfort with shells and CLI tools |
| **Version control (Git)** | IaC configurations are version-controlled; branching and PRs are central to the workflow |
| **Networking fundamentals** | Infrastructure includes VPCs, subnets, routing, DNS, load balancers, and firewalls |
| **YAML and JSON syntax** | Many IaC tools use YAML or JSON for configuration and templates |
| **A cloud provider account** | Hands-on practice requires an AWS, Azure, or GCP account (free tiers available) |

### Required Tools

Install the following tools to follow hands-on examples:

```bash
# Terraform (declarative IaC — the most widely adopted tool)
# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
# Ubuntu/Debian
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install terraform

# Pulumi (IaC with general-purpose languages)
# macOS
brew install pulumi
# Linux
curl -fsSL https://get.pulumi.com | sh

# AWS CLI (interact with AWS resources)
# macOS
brew install awscli
# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# tflint (Terraform linter)
# macOS
brew install tflint
# Linux
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# checkov (policy-as-code scanner for IaC)
pip install checkov

# Infracost (cost estimation for Terraform)
# macOS
brew install infracost
# Linux
curl -fsSL https://raw.githubusercontent.com/infracost/infracost/master/scripts/install.sh | sh

# Git (version control — required for all IaC workflows)
# macOS
brew install git
# Ubuntu/Debian
sudo apt-get install -y git

# jq (parse JSON output from IaC tools and cloud APIs)
# macOS
brew install jq
# Ubuntu/Debian
sudo apt-get install -y jq

# Verify installations
terraform version
pulumi version
aws --version
tflint --version
checkov --version
infracost --version
git --version
jq --version
```

---

## Next Steps

Work through the Infrastructure as Code series in order, or jump to the topic most relevant to you:

| File | Topic | Description |
|---|---|---|
| [01-TERRAFORM.md](01-TERRAFORM.md) | Terraform | HCL, providers, resources, modules, state backends |
| [02-PULUMI.md](02-PULUMI.md) | Pulumi | General-purpose languages for IaC, stacks, automation API |
| [03-CLOUDFORMATION.md](03-CLOUDFORMATION.md) | CloudFormation | AWS-native IaC, nested stacks, change sets, drift detection |
| [04-STATE-MANAGEMENT.md](04-STATE-MANAGEMENT.md) | State Management | Remote state, locking, import, state surgery |
| [05-MODULES-AND-REUSE.md](05-MODULES-AND-REUSE.md) | Modules & Reuse | Module design, versioning, registries |
| [06-TESTING.md](06-TESTING.md) | Testing | Policy as code, plan validation, integration testing |
| [07-SECRETS-AND-VARIABLES.md](07-SECRETS-AND-VARIABLES.md) | Secrets & Variables | Sensitive values, variable hierarchies, environments |
| [08-MULTI-CLOUD.md](08-MULTI-CLOUD.md) | Multi-Cloud | Multi-cloud patterns, abstraction layers |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Directory structure, naming, CI/CD integration |
| [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md) | Anti-Patterns | Common IaC mistakes and how to avoid them |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Structured learning guide with exercises |

### Suggested Learning Path by Role

```
Developer:
  00-OVERVIEW → 01-TERRAFORM → 04-STATE-MANAGEMENT → 06-TESTING

DevOps / Platform Engineer:
  00-OVERVIEW → 01-TERRAFORM → 05-MODULES-AND-REUSE → 09-BEST-PRACTICES → 08-MULTI-CLOUD

SRE:
  00-OVERVIEW → 04-STATE-MANAGEMENT → 07-SECRETS-AND-VARIABLES → 09-BEST-PRACTICES → 10-ANTI-PATTERNS

Architect:
  00-OVERVIEW → 08-MULTI-CLOUD → 05-MODULES-AND-REUSE → 09-BEST-PRACTICES → 10-ANTI-PATTERNS
```

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial IaC fundamentals overview documentation |
