# Modules and Reuse

A comprehensive guide to building reusable Infrastructure as Code — covering module fundamentals across Terraform, Pulumi, and CloudFormation, module design patterns, versioning strategies, registries, testing, cross-team sharing, and composition patterns for scaling IaC across organizations.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Why Modules](#why-modules)
   - [The DRY Principle in Infrastructure](#the-dry-principle-in-infrastructure)
   - [Consistency Across Environments](#consistency-across-environments)
   - [Abstraction and Encapsulation](#abstraction-and-encapsulation)
   - [Team Scalability](#team-scalability)
3. [Terraform Module Fundamentals](#terraform-module-fundamentals)
   - [Module Structure](#module-structure)
   - [Root vs Child Modules](#root-vs-child-modules)
   - [Module Sources](#module-sources)
   - [Input Variables and Outputs](#input-variables-and-outputs)
   - [Module Composition](#module-composition)
4. [Pulumi Component Resources](#pulumi-component-resources)
   - [Creating Reusable Abstractions](#creating-reusable-abstractions)
   - [ComponentResource Class](#componentresource-class)
   - [Multi-Language Components](#multi-language-components)
5. [CloudFormation Nested Stacks and Modules](#cloudformation-nested-stacks-and-modules)
   - [AWS::CloudFormation::Stack](#awscloudformationstack)
   - [CloudFormation Modules](#cloudformation-modules)
   - [Cross-Stack References](#cross-stack-references)
6. [Module Design Patterns](#module-design-patterns)
   - [Facade Pattern](#facade-pattern)
   - [Composition Over Inheritance](#composition-over-inheritance)
   - [Thin Wrapper Modules](#thin-wrapper-modules)
   - [Opinionated vs Flexible Modules](#opinionated-vs-flexible-modules)
   - [Module Interface Design](#module-interface-design)
7. [Module Versioning](#module-versioning)
   - [Semantic Versioning for Modules](#semantic-versioning-for-modules)
   - [Git Tags and Releases](#git-tags-and-releases)
   - [Terraform Registry Versioning](#terraform-registry-versioning)
   - [Version Constraints](#version-constraints)
   - [Upgrade Strategies](#upgrade-strategies)
8. [Module Registries](#module-registries)
   - [Terraform Public Registry](#terraform-public-registry)
   - [Private Registries](#private-registries)
   - [Pulumi Package Registries](#pulumi-package-registries)
9. [Module Testing](#module-testing)
   - [terraform-compliance](#terraform-compliance)
   - [Terratest](#terratest)
   - [Pulumi Unit and Integration Testing](#pulumi-unit-and-integration-testing)
10. [Cross-Team Module Sharing](#cross-team-module-sharing)
    - [Internal Module Catalogs](#internal-module-catalogs)
    - [Documentation Standards](#documentation-standards)
    - [Module Ownership](#module-ownership)
    - [Golden Modules and Blessed Configurations](#golden-modules-and-blessed-configurations)
11. [Module Composition Patterns](#module-composition-patterns)
    - [Module-of-Modules](#module-of-modules)
    - [Data Passing Between Modules](#data-passing-between-modules)
    - [Remote State as Module Output](#remote-state-as-module-output)
    - [Hub-and-Spoke Patterns](#hub-and-spoke-patterns)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **Modules and Reuse** in Infrastructure as Code — the patterns, tools, and practices that enable teams to build composable, versioned, and shareable infrastructure components. It covers module fundamentals across Terraform, Pulumi, and CloudFormation, design patterns for creating effective abstractions, versioning and registry strategies, testing approaches, cross-team sharing models, and advanced composition patterns for scaling IaC across organizations.

### Target Audience

- **DevOps Engineers** building reusable Terraform modules, Pulumi components, or CloudFormation templates for their teams
- **Platform Engineers** designing module catalogs, registries, and golden paths for self-service infrastructure
- **Site Reliability Engineers (SREs)** standardizing infrastructure patterns to ensure reliability and compliance
- **Developers** consuming infrastructure modules and needing to understand module interfaces and composition
- **Architects** evaluating module strategies for multi-team, multi-account, and enterprise-scale infrastructure

### Scope

- Why modules matter: DRY principle, consistency, abstraction, and team scalability
- Terraform module structure, sources, inputs/outputs, and composition
- Pulumi ComponentResource for reusable abstractions in general-purpose languages
- CloudFormation nested stacks, modules, and cross-stack references
- Module design patterns: facade, composition, thin wrappers, opinionated vs flexible
- Versioning strategies: semver, git tags, registry versioning, and constraints
- Module registries: public, private, and cross-tool registry options
- Testing modules: terraform-compliance, Terratest, Pulumi testing frameworks
- Cross-team sharing: catalogs, documentation, ownership, and golden modules
- Composition patterns: module-of-modules, data passing, remote state, hub-and-spoke

---

## Why Modules

### The DRY Principle in Infrastructure

Without modules, infrastructure code is duplicated across projects, environments, and teams. A VPC definition copied into 20 repositories means 20 places to update when security requirements change. The **Don't Repeat Yourself (DRY)** principle applies to infrastructure just as it does to application code.

```
Without Modules                          With Modules
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  project-a/                              modules/
  ├── vpc.tf          ← copy               └── vpc/
  ├── subnets.tf      ← copy                   ├── main.tf      ← single source
  └── security.tf     ← copy                   ├── variables.tf
                                                └── outputs.tf
  project-b/
  ├── vpc.tf          ← copy              project-a/
  ├── subnets.tf      ← copy              └── main.tf → module "vpc" { source = "../modules/vpc" }
  └── security.tf     ← copy
                                           project-b/
  project-c/                               └── main.tf → module "vpc" { source = "../modules/vpc" }
  ├── vpc.tf          ← copy
  ├── subnets.tf      ← copy              project-c/
  └── security.tf     ← copy              └── main.tf → module "vpc" { source = "../modules/vpc" }
```

The cost of duplication compounds over time:

| Problem | Without Modules | With Modules |
|---|---|---|
| **Security patch** | Update N repositories manually | Update one module, consumers upgrade |
| **Compliance change** | Audit every copy for drift | Enforce via module version |
| **Onboarding** | Read hundreds of lines per project | Understand module interface |
| **Bug fix** | Fix in every copy, hope none are missed | Fix once, release new version |

### Consistency Across Environments

Modules guarantee that development, staging, and production environments use identical infrastructure patterns. The only differences are the input variables — instance sizes, replica counts, domain names — not the structural definitions.

```hcl
# dev environment
module "app" {
  source        = "git::https://github.com/acme/modules.git//app?ref=v2.1.0"
  environment   = "dev"
  instance_type = "t3.small"
  replicas      = 1
}

# production environment
module "app" {
  source        = "git::https://github.com/acme/modules.git//app?ref=v2.1.0"
  environment   = "production"
  instance_type = "m5.xlarge"
  replicas      = 3
}
```

### Abstraction and Encapsulation

Modules hide complexity behind clean interfaces. A "database" module might create an RDS instance, parameter groups, subnet groups, security groups, IAM roles, CloudWatch alarms, and automated backups — but the consumer only sees a handful of input variables.

```
┌─────────────────────────────────────────────────┐
│  module "database"                              │
│                                                 │
│  Inputs:              Internal Resources:       │
│  ┌──────────────┐     ┌──────────────────────┐  │
│  │ engine       │────▶│ aws_db_instance       │  │
│  │ instance_class│    │ aws_db_parameter_group│  │
│  │ storage_gb   │    │ aws_db_subnet_group   │  │
│  │ multi_az     │    │ aws_security_group    │  │
│  │ backup_days  │    │ aws_iam_role          │  │
│  └──────────────┘    │ aws_cloudwatch_alarm  │  │
│                      │ aws_secretsmanager    │  │
│  Outputs:            └──────────────────────┘  │
│  ┌──────────────┐                               │
│  │ endpoint     │                               │
│  │ port         │                               │
│  │ secret_arn   │                               │
│  └──────────────┘                               │
└─────────────────────────────────────────────────┘
```

### Team Scalability

Modules enable platform teams to define infrastructure standards while allowing application teams to self-serve. Without modules, every team reinvents the wheel. With modules, platform engineers encode best practices once and application teams consume them through simple interfaces.

| Without Modules | With Modules |
|---|---|
| Every team writes VPC configs from scratch | Teams call `module "vpc"` with their parameters |
| Security reviews every unique config | Security reviews the module once |
| Knowledge siloed in individual teams | Best practices encoded in shared modules |
| Inconsistent tagging, naming, networking | Standards enforced through module logic |

---

## Terraform Module Fundamentals

### Module Structure

A Terraform module is a directory containing `.tf` files. The conventional structure separates concerns into dedicated files:

```
modules/
└── vpc/
    ├── main.tf           # Resource definitions
    ├── variables.tf      # Input variable declarations
    ├── outputs.tf        # Output value declarations
    ├── versions.tf       # Provider and Terraform version constraints
    ├── locals.tf         # Local values and computed expressions
    └── README.md         # Module documentation
```

**main.tf** — Contains the core resource definitions:

```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(var.tags, {
    Name = "${var.name}-vpc"
  })
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true

  tags = merge(var.tags, {
    Name = "${var.name}-public-${count.index + 1}"
    Tier = "public"
  })
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = merge(var.tags, {
    Name = "${var.name}-private-${count.index + 1}"
    Tier = "private"
  })
}
```

**variables.tf** — Declares all input variables with types, descriptions, defaults, and validation:

```hcl
variable "name" {
  description = "Name prefix for all resources"
  type        = string

  validation {
    condition     = length(var.name) > 0 && length(var.name) <= 32
    error_message = "Name must be between 1 and 32 characters."
  }
}

variable "cidr_block" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block."
  }
}

variable "public_subnet_cidrs" {
  description = "List of CIDR blocks for public subnets"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

variable "private_subnet_cidrs" {
  description = "List of CIDR blocks for private subnets"
  type        = list(string)
  default     = ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"]
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

**outputs.tf** — Exposes values for consumers:

```hcl
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

output "vpc_cidr_block" {
  description = "The CIDR block of the VPC"
  value       = aws_vpc.this.cidr_block
}
```

### Root vs Child Modules

Every Terraform configuration has a **root module** — the directory where you run `terraform apply`. Child modules are called from the root module using `module` blocks:

```
┌──────────────────────────────────────────────────────┐
│  Root Module (environments/production/)               │
│                                                      │
│  ┌────────────────┐  ┌────────────────┐              │
│  │ module "vpc"   │  │ module "app"   │              │
│  │ (child module) │  │ (child module) │              │
│  │                │  │                │              │
│  │ source =       │  │ source =       │              │
│  │ "../../modules │  │ "../../modules │              │
│  │  /vpc"         │  │  /app"         │              │
│  └────────┬───────┘  └────────┬───────┘              │
│           │                   │                      │
│           │  vpc_id           │                      │
│           └───────────────────┘                      │
└──────────────────────────────────────────────────────┘
```

```hcl
# environments/production/main.tf (root module)

module "vpc" {
  source = "../../modules/vpc"

  name               = "prod"
  cidr_block         = "10.0.0.0/16"
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

module "app" {
  source = "../../modules/app"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
  environment = "production"
}
```

### Module Sources

Terraform supports multiple module sources:

| Source Type | Syntax | Use Case |
|---|---|---|
| **Local path** | `source = "./modules/vpc"` | Monorepo, development |
| **Terraform Registry** | `source = "hashicorp/consul/aws"` | Public, community modules |
| **GitHub** | `source = "github.com/org/repo//modules/vpc"` | Private org modules |
| **Generic Git** | `source = "git::https://example.com/repo.git"` | Any git host |
| **S3 bucket** | `source = "s3::https://bucket.s3.amazonaws.com/vpc.zip"` | Air-gapped, artifact storage |
| **GCS bucket** | `source = "gcs::https://bucket.storage.googleapis.com/vpc.zip"` | GCP artifact storage |

```hcl
# Local path — relative to root module
module "vpc" {
  source = "../../modules/vpc"
}

# Terraform Registry — with version constraint
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}

# GitHub with ref (tag, branch, or commit)
module "vpc" {
  source = "git::https://github.com/acme/terraform-modules.git//vpc?ref=v2.1.0"
}

# S3 bucket
module "vpc" {
  source = "s3::https://my-terraform-modules.s3.us-east-1.amazonaws.com/vpc/v2.1.0.zip"
}
```

### Input Variables and Outputs

Modules communicate through a strict contract of **input variables** and **output values**. This forms the module's public API.

Best practices for module interfaces:

```hcl
# Good: typed, validated, documented variables
variable "engine_version" {
  description = "PostgreSQL engine version"
  type        = string
  default     = "15.4"

  validation {
    condition     = contains(["14.9", "15.4", "16.1"], var.engine_version)
    error_message = "Supported versions: 14.9, 15.4, 16.1"
  }
}

# Good: sensitive output for credentials
output "database_password" {
  description = "The generated database password"
  value       = random_password.db.result
  sensitive   = true
}

# Good: complex output for downstream consumers
output "connection_info" {
  description = "Database connection information"
  value = {
    host     = aws_db_instance.this.address
    port     = aws_db_instance.this.port
    database = aws_db_instance.this.db_name
    username = aws_db_instance.this.username
  }
}
```

### Module Composition

Modules call other modules to build layered abstractions:

```hcl
# modules/application/main.tf — composes lower-level modules

module "networking" {
  source = "../networking"

  vpc_cidr           = var.vpc_cidr
  availability_zones = var.availability_zones
  environment        = var.environment
}

module "database" {
  source = "../database"

  vpc_id             = module.networking.vpc_id
  subnet_ids         = module.networking.private_subnet_ids
  engine             = "postgres"
  instance_class     = var.db_instance_class
  environment        = var.environment
}

module "compute" {
  source = "../compute"

  vpc_id             = module.networking.vpc_id
  subnet_ids         = module.networking.private_subnet_ids
  database_endpoint  = module.database.endpoint
  instance_type      = var.instance_type
  min_size           = var.min_instances
  max_size           = var.max_instances
  environment        = var.environment
}
```

---

## Pulumi Component Resources

### Creating Reusable Abstractions

Pulumi uses general-purpose programming languages, so reusable abstractions are created using classes and functions rather than directory conventions. The primary mechanism is the **ComponentResource** class.

```typescript
// components/vpc.ts
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

export interface VpcArgs {
  cidrBlock: string;
  availabilityZones: string[];
  publicSubnetCidrs: string[];
  privateSubnetCidrs: string[];
  tags?: Record<string, string>;
}

export class Vpc extends pulumi.ComponentResource {
  public readonly vpcId: pulumi.Output<string>;
  public readonly publicSubnetIds: pulumi.Output<string>[];
  public readonly privateSubnetIds: pulumi.Output<string>[];

  constructor(name: string, args: VpcArgs, opts?: pulumi.ComponentResourceOptions) {
    super("custom:networking:Vpc", name, {}, opts);

    const vpc = new aws.ec2.Vpc(`${name}-vpc`, {
      cidrBlock: args.cidrBlock,
      enableDnsSupport: true,
      enableDnsHostnames: true,
      tags: { ...args.tags, Name: `${name}-vpc` },
    }, { parent: this });

    this.vpcId = vpc.id;

    this.publicSubnetIds = args.publicSubnetCidrs.map((cidr, i) => {
      const subnet = new aws.ec2.Subnet(`${name}-public-${i}`, {
        vpcId: vpc.id,
        cidrBlock: cidr,
        availabilityZone: args.availabilityZones[i],
        mapPublicIpOnLaunch: true,
        tags: { ...args.tags, Name: `${name}-public-${i + 1}`, Tier: "public" },
      }, { parent: this });
      return subnet.id;
    });

    this.privateSubnetIds = args.privateSubnetCidrs.map((cidr, i) => {
      const subnet = new aws.ec2.Subnet(`${name}-private-${i}`, {
        vpcId: vpc.id,
        cidrBlock: cidr,
        availabilityZone: args.availabilityZones[i],
        tags: { ...args.tags, Name: `${name}-private-${i + 1}`, Tier: "private" },
      }, { parent: this });
      return subnet.id;
    });

    this.registerOutputs({
      vpcId: this.vpcId,
      publicSubnetIds: this.publicSubnetIds,
      privateSubnetIds: this.privateSubnetIds,
    });
  }
}
```

Consuming the component:

```typescript
// index.ts
import { Vpc } from "./components/vpc";

const vpc = new Vpc("prod", {
  cidrBlock: "10.0.0.0/16",
  availabilityZones: ["us-east-1a", "us-east-1b", "us-east-1c"],
  publicSubnetCidrs: ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"],
  privateSubnetCidrs: ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"],
  tags: { Environment: "production", ManagedBy: "pulumi" },
});

export const vpcId = vpc.vpcId;
export const publicSubnets = vpc.publicSubnetIds;
```

### ComponentResource Class

The `ComponentResource` class is the foundation for Pulumi's reuse model:

| Concept | Description |
|---|---|
| **Type token** | Unique identifier like `custom:networking:Vpc` — used in state tracking |
| **Parent relationship** | Child resources use `{ parent: this }` to group under the component |
| **registerOutputs** | Declares the component's public outputs for state serialization |
| **ComponentResourceOptions** | Supports `providers`, `aliases`, `dependsOn`, and `protect` |

Key differences from Terraform modules:

| Feature | Terraform Module | Pulumi ComponentResource |
|---|---|---|
| **Language** | HCL | TypeScript, Python, Go, C#, Java |
| **Abstraction** | Directory with `.tf` files | Class extending ComponentResource |
| **State grouping** | Flat (prefixed by module path) | Hierarchical (parent-child tree) |
| **Logic** | Limited (count, for_each, conditions) | Full programming language constructs |
| **Testing** | External tools (Terratest) | Native unit testing frameworks |
| **Type safety** | Validation blocks | Compile-time type checking |

### Multi-Language Components

Pulumi supports authoring components in one language and consuming them in another via **Pulumi Packages**:

```
┌──────────────────────────────────────────────────────────┐
│  Multi-Language Component Architecture                    │
│                                                          │
│  ┌─────────────────────┐                                 │
│  │ Component Author     │                                │
│  │ (TypeScript/Python)  │                                │
│  └──────────┬──────────┘                                 │
│             │                                            │
│             ▼                                            │
│  ┌─────────────────────┐                                 │
│  │ Schema (schema.json) │                                │
│  └──────────┬──────────┘                                 │
│             │                                            │
│     ┌───────┼───────┬───────┐                            │
│     ▼       ▼       ▼       ▼                            │
│  ┌──────┐┌──────┐┌──────┐┌──────┐                       │
│  │  TS  ││  Py  ││  Go  ││  C#  │  Generated SDKs      │
│  │  SDK ││  SDK ││  SDK ││  SDK │                       │
│  └──────┘└──────┘└──────┘└──────┘                       │
└──────────────────────────────────────────────────────────┘
```

```typescript
// Provider implementation (TypeScript)
import * as pulumi from "@pulumi/pulumi";

export class StaticWebsite extends pulumi.ComponentResource {
  public readonly bucketName: pulumi.Output<string>;
  public readonly cdnUrl: pulumi.Output<string>;

  constructor(name: string, args: StaticWebsiteArgs, opts?: pulumi.ComponentResourceOptions) {
    super("acme:web:StaticWebsite", name, {}, opts);

    // Create S3 bucket, CloudFront distribution, Route53 records...
    // Implementation details hidden from consumers

    this.registerOutputs({
      bucketName: this.bucketName,
      cdnUrl: this.cdnUrl,
    });
  }
}
```

---

## CloudFormation Nested Stacks and Modules

### AWS::CloudFormation::Stack

CloudFormation achieves reuse through **nested stacks** — a parent template references child templates stored in S3:

```yaml
# parent-stack.yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Production application stack

Parameters:
  Environment:
    Type: String
    Default: production
    AllowedValues:
      - development
      - staging
      - production

Resources:
  NetworkingStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-templates/networking/v2.1.0/template.yaml
      Parameters:
        Environment: !Ref Environment
        VpcCidr: "10.0.0.0/16"
      Tags:
        - Key: Environment
          Value: !Ref Environment

  DatabaseStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: NetworkingStack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-templates/database/v1.3.0/template.yaml
      Parameters:
        VpcId: !GetAtt NetworkingStack.Outputs.VpcId
        SubnetIds: !GetAtt NetworkingStack.Outputs.PrivateSubnetIds
        Engine: postgres
        InstanceClass: db.r6g.xlarge

  ApplicationStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: DatabaseStack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-templates/application/v3.0.0/template.yaml
      Parameters:
        VpcId: !GetAtt NetworkingStack.Outputs.VpcId
        SubnetIds: !GetAtt NetworkingStack.Outputs.PrivateSubnetIds
        DatabaseEndpoint: !GetAtt DatabaseStack.Outputs.Endpoint

Outputs:
  ApplicationUrl:
    Value: !GetAtt ApplicationStack.Outputs.LoadBalancerDns
```

The child networking template:

```yaml
# networking/template.yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Networking module — VPC, subnets, NAT gateways

Parameters:
  Environment:
    Type: String
  VpcCidr:
    Type: String
    Default: "10.0.0.0/16"

Resources:
  Vpc:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-vpc"

  PublicSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref Vpc
      CidrBlock: "10.0.1.0/24"
      AvailabilityZone: !Select [0, !GetAZs ""]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-public-1"

Outputs:
  VpcId:
    Description: VPC identifier
    Value: !Ref Vpc
  PrivateSubnetIds:
    Description: Comma-separated private subnet IDs
    Value: !Join [",", [!Ref PrivateSubnetA, !Ref PrivateSubnetB, !Ref PrivateSubnetC]]
```

### CloudFormation Modules

CloudFormation **Modules** (introduced 2020) package reusable resource configurations registered in the CloudFormation registry:

```yaml
# Using a registered module
Resources:
  MyVpc:
    Type: Acme::Networking::Vpc::MODULE
    Properties:
      Environment: production
      VpcCidr: "10.0.0.0/16"
      AvailabilityZones:
        - us-east-1a
        - us-east-1b
        - us-east-1c
```

Module vs nested stack comparison:

| Feature | Nested Stacks | CloudFormation Modules |
|---|---|---|
| **Template location** | S3 URL | CloudFormation Registry |
| **Versioning** | S3 path-based | Registry version |
| **State visibility** | Separate stack | Inline in parent stack |
| **Update behavior** | Update nested stack | Resources updated directly |
| **Drift detection** | Per-stack | Per-resource |
| **Rollback granularity** | Per-stack | Per-resource |

### Cross-Stack References

CloudFormation supports cross-stack references through **Exports** and **Fn::ImportValue**:

```yaml
# networking-stack.yaml — exports values
Outputs:
  VpcId:
    Description: VPC ID for cross-stack reference
    Value: !Ref Vpc
    Export:
      Name: !Sub "${AWS::StackName}-VpcId"
  PrivateSubnetIds:
    Description: Private subnet IDs
    Value: !Join [",", [!Ref PrivateSubnetA, !Ref PrivateSubnetB]]
    Export:
      Name: !Sub "${AWS::StackName}-PrivateSubnetIds"
```

```yaml
# application-stack.yaml — imports values
Resources:
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Subnets: !Split [",", !ImportValue "networking-stack-PrivateSubnetIds"]
      SecurityGroups:
        - !Ref AlbSecurityGroup

  AlbSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !ImportValue "networking-stack-VpcId"
      GroupDescription: ALB security group
```

---

## Module Design Patterns

### Facade Pattern

A **facade module** wraps a complex subsystem behind a simplified interface. Consumers interact with a small number of high-level parameters while the module handles all implementation details:

```hcl
# modules/web-application/main.tf — facade over multiple AWS services

variable "name" {
  description = "Application name"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "container_image" {
  description = "Docker image URI"
  type        = string
}

variable "container_port" {
  description = "Container port"
  type        = number
  default     = 8080
}

# Internal: creates ALB, ECS service, task definition, security groups,
# CloudWatch log group, IAM roles, target groups, listener rules
resource "aws_ecs_service" "this" {
  name            = var.name
  cluster         = aws_ecs_cluster.this.id
  task_definition = aws_ecs_task_definition.this.arn
  desired_count   = var.environment == "production" ? 3 : 1

  load_balancer {
    target_group_arn = aws_lb_target_group.this.arn
    container_name   = var.name
    container_port   = var.container_port
  }
}

# Consumer only sees:
output "url" {
  value = "https://${aws_lb.this.dns_name}"
}

output "log_group" {
  value = aws_cloudwatch_log_group.this.name
}
```

### Composition Over Inheritance

Infrastructure modules cannot inherit from each other, but they can compose. Prefer building complex modules by calling simpler modules rather than duplicating resource blocks:

```
Composition Pattern
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────────────────────┐
  │  module "microservice"      │  ← High-level composition
  │                             │
  │  ┌───────────┐ ┌─────────┐ │
  │  │ module    │ │ module  │ │
  │  │ "ecs"     │ │ "alb"   │ │  ← Mid-level building blocks
  │  └─────┬─────┘ └────┬────┘ │
  │        │             │      │
  │  ┌─────┴─────┐ ┌────┴────┐ │
  │  │ module    │ │ module  │ │
  │  │ "iam"     │ │ "sg"    │ │  ← Low-level primitives
  │  └───────────┘ └─────────┘ │
  └─────────────────────────────┘
```

```hcl
# modules/microservice/main.tf

module "iam" {
  source      = "../iam-role"
  name        = var.name
  policy_arns = var.policy_arns
}

module "security_group" {
  source  = "../security-group"
  vpc_id  = var.vpc_id
  name    = var.name
  ingress_rules = [
    { port = var.container_port, cidr = var.vpc_cidr }
  ]
}

module "alb" {
  source     = "../alb"
  name       = var.name
  vpc_id     = var.vpc_id
  subnet_ids = var.public_subnet_ids
}

module "ecs_service" {
  source            = "../ecs-service"
  name              = var.name
  cluster_id        = var.cluster_id
  task_role_arn     = module.iam.role_arn
  security_group_id = module.security_group.id
  target_group_arn  = module.alb.target_group_arn
  container_image   = var.container_image
  container_port    = var.container_port
}
```

### Thin Wrapper Modules

A **thin wrapper** adds organization-specific defaults and policies around a community or provider module without duplicating its logic:

```hcl
# modules/rds-postgres/main.tf — thin wrapper around terraform-aws-modules/rds

module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"

  identifier = "${var.name}-${var.environment}"

  engine               = "postgres"
  engine_version       = var.engine_version
  family               = "postgres${split(".", var.engine_version)[0]}"
  major_engine_version = split(".", var.engine_version)[0]
  instance_class       = var.instance_class

  # Organization defaults — enforced, not configurable
  storage_encrypted       = true
  deletion_protection     = var.environment == "production"
  backup_retention_period = var.environment == "production" ? 30 : 7
  multi_az                = var.environment == "production"
  copy_tags_to_snapshot   = true

  # Passed through from consumer
  allocated_storage = var.storage_gb
  db_name           = var.database_name
  username          = var.master_username
  port              = 5432

  vpc_security_group_ids = var.security_group_ids
  subnet_ids             = var.subnet_ids

  tags = merge(var.tags, {
    Environment = var.environment
    ManagedBy   = "terraform"
    Module      = "rds-postgres"
  })
}
```

### Opinionated vs Flexible Modules

Module design exists on a spectrum from highly opinionated to fully flexible:

```
Opinionated                                              Flexible
◄──────────────────────────────────────────────────────────────────►

  "Golden module"            "Building block"         "Passthrough"
  Few inputs                 Moderate inputs          Many inputs
  Enforced standards         Sensible defaults        No defaults
  Easy to consume            Balanced                 Hard to consume
  Hard to customize          Moderate flexibility     Full control

  Example:                   Example:                 Example:
  module "app" {             module "ecs" {           module "ecs" {
    name = "orders"            name    = "orders"       name    = "orders"
    team = "commerce"          cpu     = 512            cpu     = 512
  }                            memory  = 1024           memory  = 1024
                               image   = "app:v1"       image   = "app:v1"
                             }                          task_role_arn = "..."
                                                        execution_role_arn = "..."
                                                        network_mode = "awsvpc"
                                                        # ... 30 more params
                                                      }
```

Guidelines for choosing:

| Module Type | When to Use | Target Consumer |
|---|---|---|
| **Opinionated** | Organization-wide standards, compliance requirements | Application developers |
| **Balanced** | Team-level building blocks, moderate customization needed | DevOps engineers |
| **Flexible** | Low-level primitives, maximum configurability required | Platform engineers |

### Module Interface Design

Design module interfaces like APIs — they are contracts that must remain stable:

```hcl
# Good: group related inputs with object types
variable "database_config" {
  description = "Database configuration"
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    storage_gb     = number
    multi_az       = optional(bool, false)
  })
}

# Good: use optional() with defaults (Terraform 1.3+)
variable "monitoring" {
  description = "Monitoring configuration"
  type = object({
    enabled              = optional(bool, true)
    alarm_email          = optional(string, null)
    enhanced_monitoring  = optional(bool, false)
    performance_insights = optional(bool, true)
  })
  default = {}
}

# Good: output structured data
output "cluster" {
  description = "ECS cluster information"
  value = {
    id   = aws_ecs_cluster.this.id
    name = aws_ecs_cluster.this.name
    arn  = aws_ecs_cluster.this.arn
  }
}
```

---

## Module Versioning

### Semantic Versioning for Modules

Apply **semantic versioning** (semver) to infrastructure modules just as you would to libraries:

```
MAJOR.MINOR.PATCH
  │     │     │
  │     │     └── Bug fixes, documentation (no interface changes)
  │     └──────── New features, new optional variables (backward compatible)
  └────────────── Breaking changes (removed variables, renamed outputs, resource recreation)
```

| Change | Semver Impact | Example |
|---|---|---|
| Fix a tag format bug | PATCH (1.0.0 → 1.0.1) | Correcting tag propagation |
| Add optional `monitoring` variable | MINOR (1.0.1 → 1.1.0) | New feature, existing configs unchanged |
| Rename `subnet_ids` to `private_subnet_ids` | MAJOR (1.1.0 → 2.0.0) | Consumers must update their code |
| Change resource that causes replacement | MAJOR (1.1.0 → 2.0.0) | Existing infrastructure destroyed and recreated |

### Git Tags and Releases

Use git tags as the versioning mechanism for modules stored in git repositories:

```bash
# Tag a new release
git tag -a v2.1.0 -m "feat: add optional monitoring configuration"
git push origin v2.1.0

# Reference the tag in module source
# source = "git::https://github.com/acme/modules.git//vpc?ref=v2.1.0"
```

Recommended release workflow:

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Feature  │    │  Pull    │    │  Merge   │    │  Tag     │
│  Branch   │───▶│  Request │───▶│  to main │───▶│  Release │
│           │    │  + Tests │    │          │    │  v2.1.0  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
```

### Terraform Registry Versioning

The Terraform Registry enforces semver through git tags:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.1"
}
```

Registry modules follow the naming convention `terraform-<PROVIDER>-<NAME>`:

```
terraform-aws-vpc          → registry.terraform.io/modules/hashicorp/vpc/aws
terraform-aws-eks          → registry.terraform.io/modules/terraform-aws-modules/eks/aws
terraform-google-network   → registry.terraform.io/modules/terraform-google-modules/network/google
```

### Version Constraints

Terraform supports multiple constraint operators:

| Operator | Example | Meaning |
|---|---|---|
| `=` | `version = "= 2.1.0"` | Exact version only |
| `!=` | `version = "!= 2.0.0"` | Exclude a specific version |
| `>`, `>=`, `<`, `<=` | `version = ">= 2.0, < 3.0"` | Range constraint |
| `~>` | `version = "~> 2.1"` | Pessimistic — allows 2.1.x but not 2.2.0 |

```hcl
# Recommended: pessimistic constraint for stability
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.5"
}

# Pin exact version for production stability
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "= 5.5.1"
}

# Range constraint for flexibility
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = ">= 5.0, < 6.0"
}
```

### Upgrade Strategies

Safe module upgrade workflow:

```
1. Review changelog            → Understand what changed
2. Update version constraint   → Point to new version
3. Run terraform init          → Download new module version
4. Run terraform plan          → Review planned changes
5. Check for resource replacement → CRITICAL: will anything be destroyed?
6. Apply in dev first          → Validate in non-production
7. Promote to staging          → Run integration tests
8. Apply in production         → With approval gates
```

```hcl
# Step 1: Check current version
# terraform providers lock -platform=linux_amd64

# Step 2: Update constraint
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.5"  # was "~> 5.4"
}

# Step 3: Run init and plan
# terraform init -upgrade
# terraform plan -out=upgrade.tfplan
```

---

## Module Registries

### Terraform Public Registry

The [Terraform Registry](https://registry.terraform.io/) hosts community and verified modules:

```
┌──────────────────────────────────────────────────┐
│  Terraform Registry                              │
│                                                  │
│  ┌────────────────────┐  ┌────────────────────┐  │
│  │  Verified Modules  │  │  Community Modules │  │
│  │  (HashiCorp badge) │  │  (Any publisher)   │  │
│  └────────────────────┘  └────────────────────┘  │
│                                                  │
│  Features:                                       │
│  • Automatic version detection from git tags     │
│  • Generated documentation from variables/outputs│
│  • Submodule support                             │
│  • Example configurations                        │
│  • Download counts and popularity metrics        │
└──────────────────────────────────────────────────┘
```

```hcl
# Using verified modules from the public registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "prod-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = false
}
```

### Private Registries

Organizations host private modules using multiple approaches:

| Registry Type | Setup Complexity | Features |
|---|---|---|
| **Terraform Cloud/Enterprise** | Low | Built-in private registry, VCS integration |
| **Artifactory** | Medium | Universal artifact management, fine-grained access |
| **S3 + Module Index** | Medium | Self-hosted, custom CI/CD publishing |
| **GitLab Terraform Registry** | Low | Built into GitLab, tied to project repos |
| **GitHub Packages** | Low | Integrated with GitHub repos and Actions |

Terraform Cloud private registry:

```hcl
# Configure Terraform Cloud as registry source
terraform {
  cloud {
    organization = "acme-corp"
  }
}

# Reference private module
module "vpc" {
  source  = "app.terraform.io/acme-corp/vpc/aws"
  version = "~> 3.0"
}
```

S3-based private registry:

```hcl
# Reference module from S3
module "vpc" {
  source = "s3::https://acme-terraform-modules.s3.us-east-1.amazonaws.com/vpc/v3.2.0.zip"
}
```

Publishing workflow for private registries:

```bash
#!/bin/bash
# publish-module.sh — publish a module version to S3

MODULE_NAME="vpc"
VERSION="3.2.0"
BUCKET="acme-terraform-modules"

# Package the module
cd "modules/${MODULE_NAME}"
zip -r "/tmp/${MODULE_NAME}-${VERSION}.zip" . -x "*.git*" "*.terraform*"

# Upload to S3
aws s3 cp "/tmp/${MODULE_NAME}-${VERSION}.zip" \
  "s3://${BUCKET}/${MODULE_NAME}/v${VERSION}.zip"

# Update latest pointer
echo "${VERSION}" | aws s3 cp - "s3://${BUCKET}/${MODULE_NAME}/LATEST"
```

### Pulumi Package Registries

Pulumi components are distributed as native language packages:

```typescript
// package.json for a Pulumi component package
{
  "name": "@acme/pulumi-networking",
  "version": "2.1.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "peerDependencies": {
    "@pulumi/pulumi": "^3.0.0",
    "@pulumi/aws": "^6.0.0"
  }
}
```

```bash
# Publish to private npm registry
npm publish --registry https://npm.acme.com

# Consume in a Pulumi project
npm install @acme/pulumi-networking
```

```typescript
// Consuming the published package
import { Vpc } from "@acme/pulumi-networking";

const vpc = new Vpc("production", {
  cidrBlock: "10.0.0.0/16",
  availabilityZones: ["us-east-1a", "us-east-1b"],
});
```

---

## Module Testing

### terraform-compliance

[terraform-compliance](https://terraform-compliance.com/) uses BDD-style tests to validate Terraform plans against policies:

```gherkin
# features/vpc.feature

Feature: VPC module compliance
  Modules must meet security and tagging requirements

  Scenario: VPC must have DNS support enabled
    Given I have aws_vpc defined
    Then it must have enable_dns_support
    And its value must be true

  Scenario: All subnets must be tagged
    Given I have aws_subnet defined
    Then it must have tags
    And it must contain Name

  Scenario: Private subnets must not have public IPs
    Given I have aws_subnet defined
    When its tags contain "Tier"
    And its tags "Tier" is "private"
    Then it must have map_public_ip_on_launch
    And its value must be false
```

```bash
# Run terraform-compliance against a plan
terraform plan -out=plan.tfplan
terraform show -json plan.tfplan > plan.json
terraform-compliance -p plan.json -f features/
```

### Terratest

[Terratest](https://terratest.gruntwork.io/) writes infrastructure tests in Go:

```go
// test/vpc_test.go
package test

import (
	"testing"

	"github.com/gruntwork-io/terratest/modules/aws"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestVpcModule(t *testing.T) {
	t.Parallel()

	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../modules/vpc",
		Vars: map[string]interface{}{
			"name":               "test",
			"cidr_block":         "10.99.0.0/16",
			"availability_zones": []string{"us-east-1a", "us-east-1b"},
			"public_subnet_cidrs":  []string{"10.99.1.0/24", "10.99.2.0/24"},
			"private_subnet_cidrs": []string{"10.99.11.0/24", "10.99.12.0/24"},
		},
	})

	defer terraform.Destroy(t, terraformOptions)
	terraform.InitAndApply(t, terraformOptions)

	// Validate outputs
	vpcId := terraform.Output(t, terraformOptions, "vpc_id")
	assert.NotEmpty(t, vpcId)

	publicSubnets := terraform.OutputList(t, terraformOptions, "public_subnet_ids")
	assert.Equal(t, 2, len(publicSubnets))

	privateSubnets := terraform.OutputList(t, terraformOptions, "private_subnet_ids")
	assert.Equal(t, 2, len(privateSubnets))

	// Validate VPC attributes via AWS API
	vpc := aws.GetVpcById(t, vpcId, "us-east-1")
	assert.Equal(t, "10.99.0.0/16", vpc.CidrBlock)
	assert.True(t, vpc.EnableDnsSupport)
}
```

### Pulumi Unit and Integration Testing

Pulumi supports native unit testing by mocking cloud provider calls:

```typescript
// vpc.test.ts
import * as pulumi from "@pulumi/pulumi";
import { Vpc } from "./components/vpc";

// Mock Pulumi runtime
pulumi.runtime.setMocks({
  newResource: (args: pulumi.runtime.MockResourceArgs): { id: string; state: any } => {
    return {
      id: `${args.name}-id`,
      state: args.inputs,
    };
  },
  call: (args: pulumi.runtime.MockCallArgs) => {
    return args.inputs;
  },
});

describe("Vpc Component", () => {
  let vpc: Vpc;

  beforeAll(async () => {
    vpc = new Vpc("test", {
      cidrBlock: "10.0.0.0/16",
      availabilityZones: ["us-east-1a", "us-east-1b"],
      publicSubnetCidrs: ["10.0.1.0/24", "10.0.2.0/24"],
      privateSubnetCidrs: ["10.0.11.0/24", "10.0.12.0/24"],
    });
  });

  test("VPC ID is populated", (done) => {
    pulumi.all([vpc.vpcId]).apply(([id]) => {
      expect(id).toBeDefined();
      expect(id).toContain("test");
      done();
    });
  });

  test("creates correct number of public subnets", () => {
    expect(vpc.publicSubnetIds.length).toBe(2);
  });

  test("creates correct number of private subnets", () => {
    expect(vpc.privateSubnetIds.length).toBe(2);
  });
});
```

Module testing strategy comparison:

| Approach | Speed | Coverage | Real Resources | Use Case |
|---|---|---|---|---|
| **terraform validate** | Seconds | Syntax only | No | CI gate, fast feedback |
| **terraform-compliance** | Seconds | Policy / plan | No | Compliance, security |
| **Terratest (plan only)** | Seconds | Plan assertions | No | Unit-level validation |
| **Terratest (apply)** | Minutes | Full integration | Yes | End-to-end validation |
| **Pulumi unit tests** | Seconds | Logic, structure | No | Component unit tests |
| **Pulumi integration** | Minutes | Full deployment | Yes | End-to-end validation |

---

## Cross-Team Module Sharing

### Internal Module Catalogs

Organizations maintain internal catalogs of approved modules:

```
┌──────────────────────────────────────────────────────────┐
│  Internal Module Catalog                                 │
│                                                          │
│  ┌──────────────────┐  ┌──────────────────┐              │
│  │  Networking       │  │  Compute          │             │
│  │  ├── vpc          │  │  ├── ecs-service  │             │
│  │  ├── subnet       │  │  ├── eks-cluster  │             │
│  │  ├── transit-gw   │  │  ├── lambda       │             │
│  │  └── dns          │  │  └── ec2-asg      │             │
│  └──────────────────┘  └──────────────────┘              │
│                                                          │
│  ┌──────────────────┐  ┌──────────────────┐              │
│  │  Data             │  │  Security         │             │
│  │  ├── rds-postgres │  │  ├── iam-role     │             │
│  │  ├── dynamodb     │  │  ├── kms-key      │             │
│  │  ├── elasticache  │  │  ├── waf          │             │
│  │  └── s3-bucket    │  │  └── security-group│            │
│  └──────────────────┘  └──────────────────┘              │
│                                                          │
│  Status: ● Stable  ◐ Beta  ○ Deprecated                 │
└──────────────────────────────────────────────────────────┘
```

### Documentation Standards

Every shared module should include:

```markdown
# Module: rds-postgres

## Overview
Creates a PostgreSQL RDS instance with organization defaults for encryption,
backup, monitoring, and security group configuration.

## Usage

```hcl
module "database" {
  source  = "app.terraform.io/acme-corp/rds-postgres/aws"
  version = "~> 3.0"

  name         = "orders-db"
  environment  = "production"
  instance_class = "db.r6g.large"
  storage_gb   = 100
  subnet_ids   = module.vpc.private_subnet_ids
}
```

## Inputs

| Name | Type | Required | Default | Description |
|---|---|---|---|---|
| name | string | yes | — | Database identifier |
| environment | string | yes | — | Environment name |
| instance_class | string | yes | — | RDS instance class |
| storage_gb | number | no | 50 | Allocated storage in GB |
| engine_version | string | no | "15.4" | PostgreSQL version |

## Outputs

| Name | Description |
|---|---|
| endpoint | Database endpoint address |
| port | Database port |
| secret_arn | Secrets Manager ARN for credentials |

## Requirements
- Terraform >= 1.5
- AWS Provider >= 5.0
```

### Module Ownership

Define clear ownership for every module:

| Role | Responsibility |
|---|---|
| **Module Owner** | Reviews PRs, approves releases, maintains backward compatibility |
| **Module Maintainer** | Day-to-day bug fixes, documentation updates, minor improvements |
| **Module Consumer** | Opens issues, submits PRs for features, provides feedback |
| **Platform Team** | Sets standards for module structure, testing, and publishing |

Ownership is typically tracked via a `CODEOWNERS` file:

```
# CODEOWNERS
modules/vpc/           @platform-team/networking
modules/rds-postgres/  @platform-team/data
modules/eks-cluster/   @platform-team/compute
modules/iam-role/      @security-team
```

### Golden Modules and Blessed Configurations

**Golden modules** are opinionated, organization-approved modules that enforce best practices. They are the "paved road" for teams to follow:

```hcl
# Golden module — enforces all organizational standards
module "golden_eks" {
  source  = "app.terraform.io/acme-corp/golden-eks/aws"
  version = "~> 1.0"

  # Only a few inputs — everything else is decided by the platform team
  cluster_name = "orders"
  environment  = "production"
  team         = "commerce"

  # The module internally enforces:
  # - Private endpoint only
  # - Encryption at rest and in transit
  # - Cluster logging to CloudWatch
  # - IRSA enabled
  # - Managed node groups with approved AMIs
  # - Network policies enabled
  # - Pod security standards enforced
}
```

Golden module governance:

```
┌────────────────┐     ┌────────────────┐     ┌────────────────┐
│  Platform Team │     │  Security Team │     │  Compliance    │
│  Designs       │────▶│  Reviews       │────▶│  Approves      │
│  Module        │     │  Module        │     │  Module        │
└────────────────┘     └────────────────┘     └────────────────┘
        │                                            │
        ▼                                            ▼
┌────────────────┐                          ┌────────────────┐
│  Published to  │                          │  App Teams     │
│  Registry      │─────────────────────────▶│  Consume       │
└────────────────┘                          └────────────────┘
```

---

## Module Composition Patterns

### Module-of-Modules

A **module-of-modules** composes multiple lower-level modules into a single high-level abstraction:

```hcl
# modules/platform/main.tf — composes the entire platform

module "networking" {
  source = "../networking"

  name               = var.platform_name
  cidr_block         = var.vpc_cidr
  availability_zones = var.availability_zones
  environment        = var.environment
}

module "cluster" {
  source = "../eks"

  cluster_name = var.platform_name
  vpc_id       = module.networking.vpc_id
  subnet_ids   = module.networking.private_subnet_ids
  environment  = var.environment
}

module "database" {
  source = "../rds-postgres"

  name           = "${var.platform_name}-db"
  vpc_id         = module.networking.vpc_id
  subnet_ids     = module.networking.private_subnet_ids
  instance_class = var.db_instance_class
  environment    = var.environment
}

module "monitoring" {
  source = "../monitoring"

  cluster_name      = module.cluster.cluster_name
  database_id       = module.database.instance_id
  sns_topic_arn     = var.alert_topic_arn
  environment       = var.environment
}
```

```
Module-of-Modules Architecture
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────────────────────────────────────────────┐
  │                 module "platform"                    │
  │                                                     │
  │  ┌─────────────┐  ┌────────────┐  ┌──────────────┐ │
  │  │ networking   │  │  cluster   │  │  database    │ │
  │  │             │  │            │  │              │ │
  │  │ VPC         │  │ EKS        │  │ RDS          │ │
  │  │ Subnets     │──▶ Node Groups│  │ Param Groups │ │
  │  │ NAT GW      │  │ IRSA       │  │ Backups      │ │
  │  │ Route Tables│  │ Add-ons    │  │ Monitoring   │ │
  │  └─────────────┘  └────────────┘  └──────────────┘ │
  │         │                │               │          │
  │         └────────────────┼───────────────┘          │
  │                          ▼                          │
  │                  ┌──────────────┐                   │
  │                  │  monitoring  │                   │
  │                  │  CloudWatch  │                   │
  │                  │  Dashboards  │                   │
  │                  │  Alarms      │                   │
  │                  └──────────────┘                   │
  └─────────────────────────────────────────────────────┘
```

### Data Passing Between Modules

Modules pass data through outputs and input variables. There are several patterns:

**Direct output references** — the simplest and most common:

```hcl
module "vpc" {
  source = "../modules/vpc"
  # ...
}

module "app" {
  source = "../modules/app"

  vpc_id     = module.vpc.vpc_id        # Direct reference
  subnet_ids = module.vpc.private_subnet_ids
}
```

**Local values for transformation** — when data needs reshaping between modules:

```hcl
locals {
  # Transform VPC outputs for the application module's expected format
  app_network_config = {
    vpc_id          = module.vpc.vpc_id
    subnet_ids      = module.vpc.private_subnet_ids
    security_groups = [module.vpc.default_security_group_id]
    dns_zone_id     = module.dns.zone_id
  }
}

module "app" {
  source = "../modules/app"

  network_config = local.app_network_config
}
```

**Data sources for loose coupling** — when modules are in separate state files:

```hcl
# In the application configuration (separate state)
data "aws_vpc" "main" {
  tags = {
    Name        = "prod-vpc"
    Environment = "production"
  }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
  tags = {
    Tier = "private"
  }
}

module "app" {
  source = "../modules/app"

  vpc_id     = data.aws_vpc.main.id
  subnet_ids = data.aws_subnets.private.ids
}
```

### Remote State as Module Output

Use `terraform_remote_state` to share outputs across separate Terraform configurations:

```hcl
# networking/outputs.tf — networking team manages this
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "private_subnet_ids" {
  value = module.vpc.private_subnet_ids
}
```

```hcl
# application/main.tf — application team consumes networking outputs
data "terraform_remote_state" "networking" {
  backend = "s3"

  config = {
    bucket = "acme-terraform-state"
    key    = "networking/production/terraform.tfstate"
    region = "us-east-1"
  }
}

module "app" {
  source = "../modules/app"

  vpc_id     = data.terraform_remote_state.networking.outputs.vpc_id
  subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
}
```

Data passing pattern comparison:

| Pattern | Coupling | State Boundary | Use Case |
|---|---|---|---|
| **Direct output** | Tight | Same state | Modules in same configuration |
| **Local transformation** | Tight | Same state | Data reshaping needed |
| **Data sources** | Loose | Cross-state | Tag/name-based discovery |
| **Remote state** | Medium | Cross-state | Explicit output sharing |
| **SSM Parameter Store** | Loose | Cross-state, cross-tool | Cross-tool integration |

### Hub-and-Spoke Patterns

The **hub-and-spoke** pattern uses a central shared-services configuration (hub) with per-team or per-application configurations (spokes) that consume hub outputs:

```
Hub-and-Spoke Architecture
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

                    ┌──────────────────────┐
                    │        Hub           │
                    │  (Shared Services)   │
                    │                      │
                    │  • VPC / Networking   │
                    │  • DNS               │
                    │  • IAM Baseline      │
                    │  • Logging           │
                    │  • Security Groups   │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   │   Spoke A    │  │   Spoke B    │  │   Spoke C    │
   │  (Orders)    │  │  (Payments)  │  │  (Inventory) │
   │              │  │              │  │              │
   │  • ECS Svc   │  │  • EKS Pods  │  │  • Lambda    │
   │  • RDS       │  │  • DynamoDB  │  │  • S3        │
   │  • SQS       │  │  • ElastiC.  │  │  • SQS       │
   └──────────────┘  └──────────────┘  └──────────────┘
```

Hub configuration:

```hcl
# hub/main.tf — managed by platform team
module "networking" {
  source = "app.terraform.io/acme-corp/vpc/aws"
  version = "~> 5.0"

  name               = "shared"
  cidr_block         = "10.0.0.0/16"
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# Publish outputs to SSM for loose coupling
resource "aws_ssm_parameter" "vpc_id" {
  name  = "/infrastructure/shared/vpc_id"
  type  = "String"
  value = module.networking.vpc_id
}

resource "aws_ssm_parameter" "private_subnets" {
  name  = "/infrastructure/shared/private_subnet_ids"
  type  = "StringList"
  value = join(",", module.networking.private_subnet_ids)
}
```

Spoke configuration:

```hcl
# spoke-orders/main.tf — managed by orders team
data "aws_ssm_parameter" "vpc_id" {
  name = "/infrastructure/shared/vpc_id"
}

data "aws_ssm_parameter" "private_subnets" {
  name = "/infrastructure/shared/private_subnet_ids"
}

module "orders_service" {
  source = "app.terraform.io/acme-corp/ecs-service/aws"
  version = "~> 2.0"

  name            = "orders"
  vpc_id          = data.aws_ssm_parameter.vpc_id.value
  subnet_ids      = split(",", data.aws_ssm_parameter.private_subnets.value)
  container_image = "acme/orders:v1.5.0"
  environment     = "production"
}
```

---

## Next Steps

Continue your Infrastructure as Code learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | IaC Fundamentals | Core IaC concepts, declarative vs imperative, tool landscape |
| [01-TERRAFORM.md](01-TERRAFORM.md) | Terraform | HCL fundamentals, providers, modules, state management |
| [02-PULUMI.md](02-PULUMI.md) | Pulumi | General-purpose languages for IaC, stacks, automation API |
| [03-CLOUDFORMATION.md](03-CLOUDFORMATION.md) | AWS CloudFormation | AWS-native IaC, nested stacks, drift detection |
| [04-STATE-MANAGEMENT.md](04-STATE-MANAGEMENT.md) | State Management | Remote state, locking, import, state surgery |
| [06-TESTING.md](06-TESTING.md) | IaC Testing | Policy as code, plan validation, integration testing |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Directory structure, naming, CI/CD integration |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial Modules and Reuse documentation |
