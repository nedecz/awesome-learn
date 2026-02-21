# Multi-Cloud Infrastructure as Code

Multi-cloud Infrastructure as Code (IaC) extends the principles of declarative infrastructure management across multiple cloud providers — enabling organizations to deploy, manage, and govern resources on AWS, Azure, GCP, and beyond from unified tooling and workflows. This document covers the business drivers, architecture patterns, tooling strategies, and operational practices that make multi-cloud IaC successful at scale.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Why Multi-Cloud](#why-multi-cloud)
   - [Business Drivers](#business-drivers)
   - [When Multi-Cloud Makes Sense](#when-multi-cloud-makes-sense)
   - [When Multi-Cloud Does Not Make Sense](#when-multi-cloud-does-not-make-sense)
3. [Multi-Cloud Architecture Patterns](#multi-cloud-architecture-patterns)
   - [Abstraction Layers](#abstraction-layers)
   - [Provider-Agnostic Modules](#provider-agnostic-modules)
   - [Cloud-Specific Implementations Behind Interfaces](#cloud-specific-implementations-behind-interfaces)
4. [Terraform Multi-Cloud](#terraform-multi-cloud)
   - [Using Multiple Providers](#using-multiple-providers)
   - [Provider Aliases](#provider-aliases)
   - [Conditional Resource Creation per Cloud](#conditional-resource-creation-per-cloud)
   - [Shared Modules with Cloud-Specific Implementations](#shared-modules-with-cloud-specific-implementations)
5. [Pulumi Multi-Cloud](#pulumi-multi-cloud)
   - [TypeScript Abstractions](#typescript-abstractions)
   - [Component Resources as Cloud-Agnostic Interfaces](#component-resources-as-cloud-agnostic-interfaces)
   - [Factory Patterns](#factory-patterns)
6. [Crossplane](#crossplane)
   - [Kubernetes-Native Multi-Cloud](#kubernetes-native-multi-cloud)
   - [Compositions and XRDs](#compositions-and-xrds)
   - [Claims and Provider-Agnostic Abstractions](#claims-and-provider-agnostic-abstractions)
7. [Multi-Cloud Networking](#multi-cloud-networking)
   - [VPC Peering Across Clouds](#vpc-peering-across-clouds)
   - [Transit Gateways](#transit-gateways)
   - [DNS Unification](#dns-unification)
   - [Service Mesh Across Clouds](#service-mesh-across-clouds)
8. [Multi-Cloud Identity and Access](#multi-cloud-identity-and-access)
   - [Federated Identity](#federated-identity)
   - [Cross-Cloud RBAC](#cross-cloud-rbac)
   - [Workload Identity Federation](#workload-identity-federation)
9. [Multi-Cloud State Management](#multi-cloud-state-management)
   - [Separate State per Cloud](#separate-state-per-cloud)
   - [Unified State Strategies](#unified-state-strategies)
   - [State Organization Patterns](#state-organization-patterns)
10. [Multi-Cloud CI/CD Pipelines](#multi-cloud-cicd-pipelines)
    - [Deploying to Multiple Clouds from One Pipeline](#deploying-to-multiple-clouds-from-one-pipeline)
    - [Provider Credentials Management](#provider-credentials-management)
    - [Environment-per-Cloud Patterns](#environment-per-cloud-patterns)
11. [Cost and Complexity Considerations](#cost-and-complexity-considerations)
    - [Hidden Costs of Multi-Cloud](#hidden-costs-of-multi-cloud)
    - [Operational Overhead](#operational-overhead)
    - [Skill Requirements](#skill-requirements)
    - [When to Simplify](#when-to-simplify)
12. [Multi-Cloud vs Hybrid Cloud](#multi-cloud-vs-hybrid-cloud)
    - [Definition Differences](#definition-differences)
    - [Use Cases](#use-cases)
    - [Architecture Differences](#architecture-differences)
13. [Multi-Cloud Tool Comparison](#multi-cloud-tool-comparison)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **multi-cloud Infrastructure as Code** — the strategies, patterns, and tools that enable organizations to manage infrastructure across multiple cloud providers using declarative configurations. It covers architecture patterns, Terraform and Pulumi multi-provider workflows, Crossplane's Kubernetes-native approach, networking and identity across clouds, state management, CI/CD pipelines, and the cost and complexity trade-offs teams must navigate.

### Target Audience

- **DevOps Engineers** building and maintaining infrastructure that spans AWS, Azure, GCP, or other cloud providers
- **Platform Engineers** designing self-service infrastructure platforms that abstract cloud-specific details behind unified interfaces
- **Architects** evaluating multi-cloud strategies, selecting tooling, and defining organizational standards for cross-cloud infrastructure
- **Site Reliability Engineers (SREs)** managing reliability, networking, and observability across multiple cloud environments
- **Engineering Managers** making strategic decisions about cloud adoption, vendor relationships, and infrastructure investment

### Scope

- Business drivers and trade-offs for adopting a multi-cloud strategy
- Architecture patterns for abstracting cloud-specific resources behind provider-agnostic interfaces
- Terraform multi-provider configurations, conditional resource creation, and shared module design
- Pulumi TypeScript abstractions, component resources, and factory patterns for multi-cloud
- Crossplane compositions, XRDs, and claims for Kubernetes-native multi-cloud management
- Cross-cloud networking, identity federation, and workload identity patterns
- State management strategies for multi-cloud Terraform and Pulumi deployments
- CI/CD pipeline design for deploying to multiple clouds with secure credential management
- Cost, complexity, and organizational considerations for multi-cloud adoption

---

## Why Multi-Cloud

### Business Drivers

Organizations pursue multi-cloud strategies for a variety of reasons. Understanding these drivers helps teams determine whether multi-cloud is a genuine requirement or an unnecessary source of complexity.

**Avoiding Vendor Lock-In**

The most commonly cited driver is reducing dependence on a single cloud provider. Organizations worry about:

- **Pricing leverage** — a single vendor can raise prices knowing the switching cost is high
- **Service continuity** — if a provider deprecates a service or changes terms, the business has alternatives
- **Negotiation power** — credible multi-cloud capability gives procurement teams leverage in contract negotiations

**Regulatory and Compliance Requirements**

Some industries and regions mandate specific data residency or provider requirements:

- **Data sovereignty** — certain countries require data to remain within national borders, and not every provider has regions in every country
- **Government contracts** — public sector workloads may require specific cloud certifications (FedRAMP, IL5, IRAP)
- **Industry regulations** — financial services, healthcare, and defense sectors often mandate multi-vendor strategies for resilience

**Regional Availability and Latency**

No single provider has the best coverage in every region:

- **Emerging markets** — a provider might have limited regions in Africa, South America, or Southeast Asia
- **Edge presence** — some providers offer better edge networking or CDN coverage in specific geographies
- **Latency requirements** — applications serving global users may benefit from deploying to whichever provider has the nearest region

**Best-of-Breed Services**

Different clouds excel at different things:

| Capability | Typical Leader | Reasoning |
|---|---|---|
| Machine Learning / AI | GCP | TensorFlow ecosystem, TPUs, Vertex AI |
| Enterprise Integration | Azure | Active Directory, Office 365, Dynamics |
| Broadest Service Catalog | AWS | Largest number of managed services |
| Kubernetes / Containers | GCP | GKE maturity, Knative, Istio origins |
| Data Analytics | GCP / AWS | BigQuery vs Redshift ecosystem |
| Hybrid / On-Premises | Azure | Azure Arc, Azure Stack |

**Acquisition and Organizational History**

In practice, many multi-cloud environments exist not by strategic design but by organic growth:

- **Mergers and acquisitions** — acquired companies bring their own cloud footprint
- **Team autonomy** — different departments chose different clouds independently
- **Legacy migrations** — workloads migrated at different times to different providers

### When Multi-Cloud Makes Sense

Multi-cloud is justified when there is a **concrete, measurable benefit** that outweighs the operational complexity:

1. **Regulatory mandate** — compliance requires workloads in providers with specific certifications or regions
2. **Best-of-breed requirement** — a critical workload genuinely performs better on a specific provider (e.g., ML training on GCP TPUs)
3. **Acquisition integration** — merging infrastructure from different clouds and a forced migration would be too costly or risky
4. **Disaster recovery** — the organization requires cloud-level DR beyond a single provider's multi-region capabilities
5. **Strategic negotiation** — the organization has the engineering capacity to maintain multi-cloud and uses it as pricing leverage

### When Multi-Cloud Does Not Make Sense

Multi-cloud introduces significant complexity. It does **not** make sense when:

1. **The team is small** — multi-cloud requires deep expertise across providers; small teams cannot maintain this effectively
2. **The driver is hypothetical lock-in** — "we might want to switch someday" is not a sufficient reason to double operational burden
3. **Workloads are tightly coupled to provider services** — if you use DynamoDB, Lambda, and Step Functions, abstracting to multi-cloud sacrifices all the value of those services
4. **There is no compliance requirement** — without a regulatory mandate, multi-cloud is a choice, not a necessity
5. **Cost savings are the only driver** — arbitraging cloud pricing sounds good in theory, but the engineering cost of maintaining portability usually exceeds the savings

```
┌─────────────────────────────────────────────────────────────┐
│              Multi-Cloud Decision Framework                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Is there a regulatory or compliance mandate?               │
│    YES ──► Multi-cloud is likely required                   │
│    NO  ──▼                                                  │
│                                                             │
│  Do you need best-of-breed services across providers?       │
│    YES ──► Multi-cloud may be justified                     │
│    NO  ──▼                                                  │
│                                                             │
│  Is the multi-cloud state a result of M&A?                  │
│    YES ──► Multi-cloud is a reality; manage it well         │
│    NO  ──▼                                                  │
│                                                             │
│  Do you have the engineering capacity to maintain it?       │
│    YES ──► Evaluate carefully; proceed with clear goals     │
│    NO  ──► Single-cloud is likely the better choice         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Multi-Cloud Architecture Patterns

### Abstraction Layers

The fundamental challenge of multi-cloud IaC is managing **provider-specific differences** behind a consistent interface. Abstraction layers sit between your application's infrastructure requirements and the cloud-specific implementations:

```
┌──────────────────────────────────────────────────────────────────┐
│                      Application Layer                           │
│           "I need a database, a queue, and a load balancer"      │
├──────────────────────────────────────────────────────────────────┤
│                     Abstraction Layer                             │
│      Translates generic requirements to cloud-specific resources │
├──────────────┬──────────────────┬────────────────────────────────┤
│     AWS      │      Azure       │          GCP                   │
│  ┌────────┐  │  ┌────────────┐  │  ┌─────────────────────┐      │
│  │  RDS   │  │  │ Azure SQL  │  │  │  Cloud SQL          │      │
│  │  SQS   │  │  │ Service Bus│  │  │  Pub/Sub            │      │
│  │  ALB   │  │  │ App GW     │  │  │  Cloud Load Balancer│      │
│  └────────┘  │  └────────────┘  │  └─────────────────────┘      │
└──────────────┴──────────────────┴────────────────────────────────┘
```

The abstraction layer can live at different levels:

| Abstraction Level | Example | Trade-off |
|---|---|---|
| **IaC Module Interface** | Terraform module with cloud-specific sub-modules | Most control, most maintenance |
| **Language Type System** | Pulumi component resources with interfaces | Strong typing, language-specific |
| **Kubernetes API** | Crossplane XRDs and compositions | Kubernetes-native, limited to K8s users |
| **Platform API** | Internal developer platform with API | Maximum abstraction, highest build cost |

### Provider-Agnostic Modules

A provider-agnostic module defines **inputs and outputs** that describe infrastructure intent without referencing any specific cloud provider:

```
┌──────────────────────────────────────────┐
│         Module: "compute-instance"        │
│                                          │
│  Inputs:                                 │
│    - name: string                        │
│    - instance_size: "small"|"medium"|    │
│                     "large"              │
│    - cloud: "aws"|"azure"|"gcp"          │
│    - region: string                      │
│                                          │
│  Outputs:                                │
│    - instance_id: string                 │
│    - public_ip: string                   │
│    - private_ip: string                  │
│                                          │
│  Internal routing:                       │
│    cloud == "aws"   → aws_instance       │
│    cloud == "azure" → azurerm_linux_vm   │
│    cloud == "gcp"   → google_compute_    │
│                       instance           │
└──────────────────────────────────────────┘
```

The key principle: **callers interact with a uniform interface** and never reference provider-specific resource types directly.

### Cloud-Specific Implementations Behind Interfaces

Each cloud-specific implementation conforms to the same interface contract. This pattern separates the **what** (a compute instance with certain properties) from the **how** (AWS EC2 vs Azure VM vs GCP Compute Engine):

```
infrastructure/
├── modules/
│   ├── compute/
│   │   ├── interface.tf          # Variable and output definitions
│   │   ├── aws/
│   │   │   └── main.tf           # aws_instance resource
│   │   ├── azure/
│   │   │   └── main.tf           # azurerm_linux_virtual_machine
│   │   └── gcp/
│   │       └── main.tf           # google_compute_instance
│   ├── database/
│   │   ├── interface.tf
│   │   ├── aws/
│   │   │   └── main.tf           # aws_rds_instance
│   │   ├── azure/
│   │   │   └── main.tf           # azurerm_mssql_server
│   │   └── gcp/
│   │       └── main.tf           # google_sql_database_instance
│   └── networking/
│       ├── interface.tf
│       ├── aws/
│       │   └── main.tf           # aws_vpc, aws_subnet
│       ├── azure/
│       │   └── main.tf           # azurerm_virtual_network
│       └── gcp/
│           └── main.tf           # google_compute_network
└── environments/
    ├── production-aws/
    │   └── main.tf
    ├── production-azure/
    │   └── main.tf
    └── production-gcp/
        └── main.tf
```

This structure allows teams to:

- **Add a new cloud** by implementing the interface contract without modifying existing code
- **Test each implementation** independently with provider-specific integration tests
- **Share configuration logic** (naming conventions, tagging) across all implementations

---

## Terraform Multi-Cloud

### Using Multiple Providers

Terraform's provider plugin architecture makes it straightforward to manage resources across multiple clouds in a single configuration. Each provider is declared independently:

```hcl
# providers.tf — declaring multiple cloud providers

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
  }
}

provider "aws" {
  region = "us-east-1"
}

provider "azurerm" {
  features {}
  subscription_id = var.azure_subscription_id
}

provider "google" {
  project = var.gcp_project_id
  region  = "us-central1"
}
```

With all three providers configured, you can create resources across clouds in the same Terraform root module:

```hcl
# AWS S3 bucket for primary storage
resource "aws_s3_bucket" "data" {
  bucket = "myapp-data-primary"
}

# Azure Blob Storage for DR replication
resource "azurerm_storage_account" "dr" {
  name                     = "myappdatarepl"
  resource_group_name      = azurerm_resource_group.dr.name
  location                 = "westeurope"
  account_tier             = "Standard"
  account_replication_type = "GRS"
}

# GCP Cloud Storage for analytics
resource "google_storage_bucket" "analytics" {
  name     = "myapp-analytics-data"
  location = "US"

  uniform_bucket_level_access = true
}
```

### Provider Aliases

When you need multiple configurations of the same provider (e.g., deploying to multiple AWS regions), use **provider aliases**:

```hcl
# Primary AWS region
provider "aws" {
  region = "us-east-1"
  alias  = "us_east"
}

# Secondary AWS region for DR
provider "aws" {
  region = "eu-west-1"
  alias  = "eu_west"
}

# Primary GCP region
provider "google" {
  project = var.gcp_project_id
  region  = "us-central1"
  alias   = "us_central"
}

# Secondary GCP region
provider "google" {
  project = var.gcp_project_id
  region  = "europe-west1"
  alias   = "eu_west"
}

# Resources reference the specific provider alias
resource "aws_instance" "web_us" {
  provider      = aws.us_east
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.medium"
}

resource "aws_instance" "web_eu" {
  provider      = aws.eu_west
  ami           = "ami-0fedcba0987654321"
  instance_type = "t3.medium"
}
```

### Conditional Resource Creation per Cloud

Use Terraform variables and conditional expressions to deploy resources to specific clouds based on configuration:

```hcl
variable "enabled_clouds" {
  description = "List of cloud providers to deploy to"
  type        = list(string)
  default     = ["aws", "gcp"]

  validation {
    condition = alltrue([
      for cloud in var.enabled_clouds : contains(["aws", "azure", "gcp"], cloud)
    ])
    error_message = "Supported clouds are: aws, azure, gcp."
  }
}

# Deploy AWS resources only if AWS is in the enabled list
resource "aws_vpc" "main" {
  count      = contains(var.enabled_clouds, "aws") ? 1 : 0
  cidr_block = "10.0.0.0/16"

  tags = {
    Name        = "main-vpc"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# Deploy Azure resources only if Azure is in the enabled list
resource "azurerm_virtual_network" "main" {
  count               = contains(var.enabled_clouds, "azure") ? 1 : 0
  name                = "main-vnet"
  address_space       = ["10.1.0.0/16"]
  location            = "eastus"
  resource_group_name = azurerm_resource_group.main[0].name
}

# Deploy GCP resources only if GCP is in the enabled list
resource "google_compute_network" "main" {
  count                   = contains(var.enabled_clouds, "gcp") ? 1 : 0
  name                    = "main-network"
  auto_create_subnetworks = false
}
```

### Shared Modules with Cloud-Specific Implementations

A common pattern is a wrapper module that delegates to cloud-specific child modules:

```hcl
# modules/database/main.tf — cloud-routing wrapper module

variable "cloud" {
  description = "Target cloud provider"
  type        = string

  validation {
    condition     = contains(["aws", "azure", "gcp"], var.cloud)
    error_message = "Supported values: aws, azure, gcp."
  }
}

variable "name" {
  description = "Database instance name"
  type        = string
}

variable "engine" {
  description = "Database engine (postgres, mysql)"
  type        = string
  default     = "postgres"
}

variable "size" {
  description = "Instance size class"
  type        = string
  default     = "small"
}

# Route to the AWS implementation
module "aws" {
  source = "./aws"
  count  = var.cloud == "aws" ? 1 : 0

  name   = var.name
  engine = var.engine
  size   = var.size
}

# Route to the Azure implementation
module "azure" {
  source = "./azure"
  count  = var.cloud == "azure" ? 1 : 0

  name   = var.name
  engine = var.engine
  size   = var.size
}

# Route to the GCP implementation
module "gcp" {
  source = "./gcp"
  count  = var.cloud == "gcp" ? 1 : 0

  name   = var.name
  engine = var.engine
  size   = var.size
}

# Unified outputs regardless of cloud
output "connection_string" {
  value = coalesce(
    try(module.aws[0].connection_string, ""),
    try(module.azure[0].connection_string, ""),
    try(module.gcp[0].connection_string, "")
  )
}

output "instance_id" {
  value = coalesce(
    try(module.aws[0].instance_id, ""),
    try(module.azure[0].instance_id, ""),
    try(module.gcp[0].instance_id, "")
  )
}
```

```hcl
# modules/database/aws/main.tf — AWS-specific implementation

variable "name" { type = string }
variable "engine" { type = string }
variable "size" { type = string }

locals {
  size_map = {
    small  = "db.t3.micro"
    medium = "db.t3.medium"
    large  = "db.r5.large"
  }
}

resource "aws_db_instance" "this" {
  identifier     = var.name
  engine         = var.engine
  instance_class = local.size_map[var.size]
  allocated_storage = 20

  skip_final_snapshot = true

  tags = {
    Name      = var.name
    ManagedBy = "terraform"
  }
}

output "connection_string" {
  value = aws_db_instance.this.endpoint
}

output "instance_id" {
  value = aws_db_instance.this.id
}
```

Callers use the wrapper module without knowing which cloud implementation runs underneath:

```hcl
# environments/production/main.tf

module "primary_database" {
  source = "../../modules/database"

  cloud  = "aws"
  name   = "myapp-primary"
  engine = "postgres"
  size   = "medium"
}

module "dr_database" {
  source = "../../modules/database"

  cloud  = "gcp"
  name   = "myapp-dr"
  engine = "postgres"
  size   = "medium"
}
```

---

## Pulumi Multi-Cloud

### TypeScript Abstractions

Pulumi's use of general-purpose programming languages (TypeScript, Python, Go, C#) provides native language features for multi-cloud abstraction — interfaces, generics, inheritance, and dependency injection:

```typescript
// types.ts — cloud-agnostic interface definitions

export interface DatabaseConfig {
  name: string;
  engine: "postgres" | "mysql";
  size: "small" | "medium" | "large";
  region: string;
}

export interface DatabaseResult {
  connectionString: pulumi.Output<string>;
  instanceId: pulumi.Output<string>;
  provider: string;
}

export interface ComputeConfig {
  name: string;
  size: "small" | "medium" | "large";
  region: string;
  imageId?: string;
}

export interface ComputeResult {
  instanceId: pulumi.Output<string>;
  publicIp: pulumi.Output<string>;
  privateIp: pulumi.Output<string>;
  provider: string;
}
```

```typescript
// aws-database.ts — AWS implementation

import * as aws from "@pulumi/aws";
import * as pulumi from "@pulumi/pulumi";
import { DatabaseConfig, DatabaseResult } from "./types";

const sizeMap: Record<string, string> = {
  small: "db.t3.micro",
  medium: "db.t3.medium",
  large: "db.r5.large",
};

export function createAwsDatabase(config: DatabaseConfig): DatabaseResult {
  const instance = new aws.rds.Instance(`${config.name}-rds`, {
    identifier: config.name,
    engine: config.engine,
    instanceClass: sizeMap[config.size],
    allocatedStorage: 20,
    skipFinalSnapshot: true,
    tags: {
      Name: config.name,
      ManagedBy: "pulumi",
    },
  });

  return {
    connectionString: instance.endpoint,
    instanceId: instance.id,
    provider: "aws",
  };
}
```

```typescript
// gcp-database.ts — GCP implementation

import * as gcp from "@pulumi/gcp";
import * as pulumi from "@pulumi/pulumi";
import { DatabaseConfig, DatabaseResult } from "./types";

const tierMap: Record<string, string> = {
  small: "db-f1-micro",
  medium: "db-custom-2-7680",
  large: "db-custom-4-15360",
};

export function createGcpDatabase(config: DatabaseConfig): DatabaseResult {
  const instance = new gcp.sql.DatabaseInstance(`${config.name}-sql`, {
    name: config.name,
    databaseVersion: config.engine === "postgres" ? "POSTGRES_15" : "MYSQL_8_0",
    region: config.region,
    settings: {
      tier: tierMap[config.size],
    },
    deletionProtection: false,
  });

  return {
    connectionString: instance.connectionName,
    instanceId: instance.id,
    provider: "gcp",
  };
}
```

### Component Resources as Cloud-Agnostic Interfaces

Pulumi **Component Resources** provide a way to create higher-level, reusable abstractions that bundle multiple cloud resources behind a single interface:

```typescript
// multi-cloud-database.ts — component resource

import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import * as gcp from "@pulumi/gcp";

export interface MultiCloudDatabaseArgs {
  cloud: "aws" | "gcp" | "azure";
  name: string;
  engine: "postgres" | "mysql";
  size: "small" | "medium" | "large";
}

export class MultiCloudDatabase extends pulumi.ComponentResource {
  public readonly connectionString: pulumi.Output<string>;
  public readonly instanceId: pulumi.Output<string>;
  public readonly cloud: string;

  constructor(
    name: string,
    args: MultiCloudDatabaseArgs,
    opts?: pulumi.ComponentResourceOptions
  ) {
    super("custom:infrastructure:MultiCloudDatabase", name, {}, opts);

    const childOpts = { parent: this };

    switch (args.cloud) {
      case "aws":
        const rds = new aws.rds.Instance(`${name}-rds`, {
          identifier: args.name,
          engine: args.engine,
          instanceClass: this.awsSizeMap(args.size),
          allocatedStorage: 20,
          skipFinalSnapshot: true,
        }, childOpts);
        this.connectionString = rds.endpoint;
        this.instanceId = rds.id;
        break;

      case "gcp":
        const sql = new gcp.sql.DatabaseInstance(`${name}-sql`, {
          name: args.name,
          databaseVersion: args.engine === "postgres"
            ? "POSTGRES_15"
            : "MYSQL_8_0",
          settings: { tier: this.gcpSizeMap(args.size) },
          deletionProtection: false,
        }, childOpts);
        this.connectionString = sql.connectionName;
        this.instanceId = sql.id;
        break;

      default:
        throw new Error(`Unsupported cloud: ${args.cloud}`);
    }

    this.cloud = args.cloud;
    this.registerOutputs({
      connectionString: this.connectionString,
      instanceId: this.instanceId,
    });
  }

  private awsSizeMap(size: string): string {
    const map: Record<string, string> = {
      small: "db.t3.micro",
      medium: "db.t3.medium",
      large: "db.r5.large",
    };
    return map[size];
  }

  private gcpSizeMap(size: string): string {
    const map: Record<string, string> = {
      small: "db-f1-micro",
      medium: "db-custom-2-7680",
      large: "db-custom-4-15360",
    };
    return map[size];
  }
}
```

Usage is clean and consistent regardless of target cloud:

```typescript
// index.ts — consuming the component resource

import { MultiCloudDatabase } from "./multi-cloud-database";

const primaryDb = new MultiCloudDatabase("primary", {
  cloud: "aws",
  name: "myapp-primary",
  engine: "postgres",
  size: "medium",
});

const drDb = new MultiCloudDatabase("dr", {
  cloud: "gcp",
  name: "myapp-dr",
  engine: "postgres",
  size: "medium",
});

export const primaryConnectionString = primaryDb.connectionString;
export const drConnectionString = drDb.connectionString;
```

### Factory Patterns

The factory pattern centralizes cloud selection logic, making it easy to add new providers without modifying consuming code:

```typescript
// database-factory.ts

import { DatabaseConfig, DatabaseResult } from "./types";
import { createAwsDatabase } from "./aws-database";
import { createGcpDatabase } from "./gcp-database";
import { createAzureDatabase } from "./azure-database";

type CloudProvider = "aws" | "gcp" | "azure";

type DatabaseFactory = (config: DatabaseConfig) => DatabaseResult;

const factories: Record<CloudProvider, DatabaseFactory> = {
  aws: createAwsDatabase,
  gcp: createGcpDatabase,
  azure: createAzureDatabase,
};

export function createDatabase(
  cloud: CloudProvider,
  config: DatabaseConfig
): DatabaseResult {
  const factory = factories[cloud];
  if (!factory) {
    throw new Error(`No database factory registered for cloud: ${cloud}`);
  }
  return factory(config);
}
```

```typescript
// index.ts — using the factory

import { createDatabase } from "./database-factory";

const config = new pulumi.Config();
const targetCloud = config.require("targetCloud") as "aws" | "gcp" | "azure";

const database = createDatabase(targetCloud, {
  name: "myapp-db",
  engine: "postgres",
  size: "medium",
  region: "us-east-1",
});

export const connectionString = database.connectionString;
```

---

## Crossplane

### Kubernetes-Native Multi-Cloud

**Crossplane** extends Kubernetes with the ability to provision and manage cloud infrastructure using the Kubernetes API. Instead of using HCL or a programming language, teams define infrastructure as Kubernetes custom resources:

```
┌────────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                       │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │               Crossplane Controllers                  │  │
│  │                                                       │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │  │
│  │  │ AWS      │  │ Azure    │  │ GCP              │   │  │
│  │  │ Provider │  │ Provider │  │ Provider         │   │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────────────┘   │  │
│  │       │              │              │                  │  │
│  └───────┼──────────────┼──────────────┼─────────────────┘  │
│          │              │              │                     │
│  ┌───────┼──────────────┼──────────────┼─────────────────┐  │
│  │       ▼              ▼              ▼                  │  │
│  │  Compositions (XRDs + Compositions)                   │  │
│  │                                                       │  │
│  │  Claims (XRCs) ── namespace-scoped requests           │  │
│  └───────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
          │              │              │
          ▼              ▼              ▼
     ┌─────────┐   ┌─────────┐   ┌─────────┐
     │  AWS    │   │  Azure  │   │  GCP    │
     │  Cloud  │   │  Cloud  │   │  Cloud  │
     └─────────┘   └─────────┘   └─────────┘
```

### Compositions and XRDs

**Composite Resource Definitions (XRDs)** define the schema for your cloud-agnostic abstraction. **Compositions** define how that abstraction maps to cloud-specific resources:

```yaml
# xrd-database.yaml — define the cloud-agnostic API

apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.infrastructure.example.com
spec:
  group: infrastructure.example.com
  names:
    kind: XDatabase
    plural: xdatabases
  claimNames:
    kind: Database
    plural: databases
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    engine:
                      type: string
                      enum: ["postgres", "mysql"]
                    size:
                      type: string
                      enum: ["small", "medium", "large"]
                    region:
                      type: string
                    cloud:
                      type: string
                      enum: ["aws", "azure", "gcp"]
                  required:
                    - engine
                    - size
                    - region
                    - cloud
```

```yaml
# composition-aws.yaml — AWS-specific composition

apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xdatabases-aws
  labels:
    cloud: aws
spec:
  compositeTypeRef:
    apiVersion: infrastructure.example.com/v1alpha1
    kind: XDatabase
  patchSets:
    - name: common-tags
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: metadata.name
          toFieldPath: spec.forProvider.tags.Name
  resources:
    - name: rds-instance
      base:
        apiVersion: rds.aws.upbound.io/v1beta1
        kind: Instance
        spec:
          forProvider:
            engine: postgres
            instanceClass: db.t3.micro
            allocatedStorage: 20
            skipFinalSnapshot: true
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.engine
          toFieldPath: spec.forProvider.engine
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.size
          toFieldPath: spec.forProvider.instanceClass
          transforms:
            - type: map
              map:
                small: db.t3.micro
                medium: db.t3.medium
                large: db.r5.large
```

```yaml
# composition-gcp.yaml — GCP-specific composition

apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xdatabases-gcp
  labels:
    cloud: gcp
spec:
  compositeTypeRef:
    apiVersion: infrastructure.example.com/v1alpha1
    kind: XDatabase
  resources:
    - name: sql-instance
      base:
        apiVersion: sql.gcp.upbound.io/v1beta1
        kind: DatabaseInstance
        spec:
          forProvider:
            databaseVersion: POSTGRES_15
            settings:
              - tier: db-f1-micro
            deletionProtection: false
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.engine
          toFieldPath: spec.forProvider.databaseVersion
          transforms:
            - type: map
              map:
                postgres: POSTGRES_15
                mysql: MYSQL_8_0
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.size
          toFieldPath: spec.forProvider.settings[0].tier
          transforms:
            - type: map
              map:
                small: db-f1-micro
                medium: db-custom-2-7680
                large: db-custom-4-15360
        - type: FromCompositeFieldPath
          fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.region
```

### Claims and Provider-Agnostic Abstractions

**Claims (XRCs)** are the namespace-scoped resources that developers actually create. They reference the XRD and Crossplane selects the appropriate composition:

```yaml
# claim-aws.yaml — developer requests a database on AWS

apiVersion: infrastructure.example.com/v1alpha1
kind: Database
metadata:
  name: myapp-primary
  namespace: team-backend
spec:
  parameters:
    engine: postgres
    size: medium
    region: us-east-1
    cloud: aws
  compositionSelector:
    matchLabels:
      cloud: aws
```

```yaml
# claim-gcp.yaml — developer requests a database on GCP

apiVersion: infrastructure.example.com/v1alpha1
kind: Database
metadata:
  name: myapp-analytics
  namespace: team-data
spec:
  parameters:
    engine: postgres
    size: large
    region: us-central1
    cloud: gcp
  compositionSelector:
    matchLabels:
      cloud: gcp
```

The developer experience is identical regardless of the target cloud — the only difference is the `cloud` parameter and the `compositionSelector`.

---

## Multi-Cloud Networking

### VPC Peering Across Clouds

Connecting networks across cloud providers requires either **VPN tunnels** or **dedicated interconnects**. The most common approach for IaC-managed multi-cloud networking is site-to-site VPN:

```hcl
# aws-vpn.tf — AWS side of the VPN tunnel to GCP

resource "aws_vpn_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "aws-to-gcp-vpn-gateway"
  }
}

resource "aws_customer_gateway" "gcp" {
  bgp_asn    = 65000
  ip_address = google_compute_address.vpn_static_ip.address
  type       = "ipsec.1"

  tags = {
    Name = "gcp-customer-gateway"
  }
}

resource "aws_vpn_connection" "to_gcp" {
  vpn_gateway_id      = aws_vpn_gateway.main.id
  customer_gateway_id = aws_customer_gateway.gcp.id
  type                = "ipsec.1"
  static_routes_only  = true

  tags = {
    Name = "aws-to-gcp-vpn"
  }
}

resource "aws_vpn_connection_route" "gcp_network" {
  destination_cidr_block = "10.128.0.0/16"
  vpn_connection_id      = aws_vpn_connection.to_gcp.id
}
```

```hcl
# gcp-vpn.tf — GCP side of the VPN tunnel to AWS

resource "google_compute_address" "vpn_static_ip" {
  name   = "vpn-static-ip"
  region = "us-central1"
}

resource "google_compute_vpn_gateway" "main" {
  name    = "gcp-to-aws-vpn-gateway"
  network = google_compute_network.main.id
  region  = "us-central1"
}

resource "google_compute_forwarding_rule" "esp" {
  name        = "vpn-esp"
  ip_protocol = "ESP"
  ip_address  = google_compute_address.vpn_static_ip.address
  target      = google_compute_vpn_gateway.main.self_link
  region      = "us-central1"
}

resource "google_compute_vpn_tunnel" "to_aws" {
  name               = "gcp-to-aws-tunnel"
  peer_ip            = aws_vpn_connection.to_gcp.tunnel1_address
  shared_secret      = aws_vpn_connection.to_gcp.tunnel1_preshared_key
  target_vpn_gateway = google_compute_vpn_gateway.main.self_link
  local_traffic_selector  = ["10.128.0.0/16"]
  remote_traffic_selector = ["10.0.0.0/16"]
  region             = "us-central1"
}

resource "google_compute_route" "to_aws" {
  name                = "route-to-aws"
  network             = google_compute_network.main.id
  dest_range          = "10.0.0.0/16"
  priority            = 1000
  next_hop_vpn_tunnel = google_compute_vpn_tunnel.to_aws.self_link
}
```

### Transit Gateways

For larger multi-cloud deployments, a **hub-and-spoke topology** using transit gateways provides centralized routing:

```
┌────────────────────────────────────────────────────────────┐
│                    Transit Hub                             │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           AWS Transit Gateway                         │  │
│  │                                                       │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────────────────┐  │  │
│  │  │ VPC A   │  │ VPC B   │  │ VPN to Azure / GCP  │  │  │
│  │  │10.0.0/16│  │10.1.0/16│  │                     │  │  │
│  │  └─────────┘  └─────────┘  └──────────┬──────────┘  │  │
│  │                                        │              │  │
│  └────────────────────────────────────────┼──────────────┘  │
│                                           │                 │
└───────────────────────────────────────────┼─────────────────┘
                                            │
                    ┌───────────────────────┼──────────────┐
                    │                       │              │
              ┌─────▼─────┐          ┌──────▼──────┐
              │   Azure   │          │    GCP      │
              │  VPN GW   │          │  VPN GW     │
              │           │          │             │
              │ VNet      │          │ VPC         │
              │10.2.0/16  │          │10.128.0/16  │
              └───────────┘          └─────────────┘
```

```hcl
# transit-gateway.tf — AWS Transit Gateway as the hub

resource "aws_ec2_transit_gateway" "hub" {
  description                     = "Multi-cloud transit hub"
  default_route_table_association = "enable"
  default_route_table_propagation = "enable"
  dns_support                     = "enable"

  tags = {
    Name = "multi-cloud-hub"
  }
}

resource "aws_ec2_transit_gateway_vpc_attachment" "vpc_a" {
  subnet_ids         = aws_subnet.vpc_a_private[*].id
  transit_gateway_id = aws_ec2_transit_gateway.hub.id
  vpc_id             = aws_vpc.a.id

  tags = {
    Name = "vpc-a-attachment"
  }
}

resource "aws_ec2_transit_gateway_vpc_attachment" "vpc_b" {
  subnet_ids         = aws_subnet.vpc_b_private[*].id
  transit_gateway_id = aws_ec2_transit_gateway.hub.id
  vpc_id             = aws_vpc.b.id

  tags = {
    Name = "vpc-b-attachment"
  }
}

# VPN attachment connects the transit gateway to Azure/GCP
resource "aws_vpn_connection" "to_azure" {
  customer_gateway_id = aws_customer_gateway.azure.id
  transit_gateway_id  = aws_ec2_transit_gateway.hub.id
  type                = "ipsec.1"

  tags = {
    Name = "tgw-to-azure-vpn"
  }
}
```

### DNS Unification

A unified DNS strategy ensures services can discover each other across clouds:

```hcl
# dns-unification.tf — Route 53 as the unified DNS layer

resource "aws_route53_zone" "internal" {
  name = "internal.mycompany.com"

  vpc {
    vpc_id = aws_vpc.main.id
  }
}

# AWS services resolve directly
resource "aws_route53_record" "api_aws" {
  zone_id = aws_route53_zone.internal.zone_id
  name    = "api.internal.mycompany.com"
  type    = "A"

  alias {
    name                   = aws_lb.api.dns_name
    zone_id                = aws_lb.api.zone_id
    evaluate_target_health = true
  }
}

# GCP services resolve via their external IP or interconnect IP
resource "aws_route53_record" "analytics_gcp" {
  zone_id = aws_route53_zone.internal.zone_id
  name    = "analytics.internal.mycompany.com"
  type    = "A"
  ttl     = 300
  records = [google_compute_address.analytics_lb.address]
}

# Azure services resolve via their interconnect IP
resource "aws_route53_record" "auth_azure" {
  zone_id = aws_route53_zone.internal.zone_id
  name    = "auth.internal.mycompany.com"
  type    = "A"
  ttl     = 300
  records = [azurerm_public_ip.auth_lb.ip_address]
}
```

### Service Mesh Across Clouds

For microservice communication across clouds, a **service mesh** provides consistent observability, traffic management, and security:

```
┌───────────────────────┐     ┌───────────────────────┐
│       AWS EKS         │     │       GCP GKE         │
│                       │     │                       │
│  ┌─────────────────┐  │     │  ┌─────────────────┐  │
│  │  Istio Control  │◄─┼─────┼──│  Istio Control  │  │
│  │  Plane (Primary)│  │     │  │  Plane (Remote) │  │
│  └────────┬────────┘  │     │  └────────┬────────┘  │
│           │           │     │           │           │
│  ┌────────▼────────┐  │     │  ┌────────▼────────┐  │
│  │  Service A      │  │     │  │  Service B      │  │
│  │  (with sidecar) │◄─┼─mTLS┼──│  (with sidecar) │  │
│  └─────────────────┘  │     │  └─────────────────┘  │
└───────────────────────┘     └───────────────────────┘
```

---

## Multi-Cloud Identity and Access

### Federated Identity

Federated identity allows a single identity provider (IdP) to authenticate users and services across multiple cloud providers:

```
┌──────────────────────────────────────────────────────────┐
│               Identity Provider (IdP)                    │
│          (Azure AD / Okta / Google Workspace)            │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐  │
│  │  SAML / OIDC │  │  SAML / OIDC  │  │  SAML / OIDC │  │
│  │  to AWS IAM  │  │  to Azure AD  │  │  to GCP IAM  │  │
│  │  Identity    │  │               │  │  Workforce    │  │
│  │  Center      │  │               │  │  Identity     │  │
│  └──────┬───────┘  └──────┬────────┘  └──────┬───────┘  │
│         │                 │                   │          │
└─────────┼─────────────────┼───────────────────┼──────────┘
          │                 │                   │
          ▼                 ▼                   ▼
     ┌─────────┐      ┌─────────┐        ┌─────────┐
     │   AWS   │      │  Azure  │        │   GCP   │
     └─────────┘      └─────────┘        └─────────┘
```

### Cross-Cloud RBAC

Define consistent role mappings across clouds using Terraform:

```hcl
# cross-cloud-rbac.tf — consistent role definitions

variable "developer_group_id" {
  description = "Identity provider group ID for developers"
  type        = string
}

variable "platform_group_id" {
  description = "Identity provider group ID for platform engineers"
  type        = string
}

# AWS — developer role
resource "aws_iam_role" "developer" {
  name = "developer"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = "arn:aws:iam::saml-provider/corporate-idp"
        }
        Action = "sts:AssumeRoleWithSAML"
        Condition = {
          StringEquals = {
            "SAML:aud" = "https://signin.aws.amazon.com/saml"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "developer_readonly" {
  role       = aws_iam_role.developer.name
  policy_arn = "arn:aws:iam::policy/ReadOnlyAccess"
}

# Azure — developer role assignment
resource "azurerm_role_assignment" "developer" {
  scope                = data.azurerm_subscription.current.id
  role_definition_name = "Reader"
  principal_id         = var.developer_group_id
}

# GCP — developer role binding
resource "google_project_iam_member" "developer" {
  project = var.gcp_project_id
  role    = "roles/viewer"
  member  = "group:developers@mycompany.com"
}
```

### Workload Identity Federation

**Workload Identity Federation** allows services running in one cloud to authenticate to another cloud without long-lived credentials:

```hcl
# gcp-to-aws.tf — GCP workload accessing AWS resources

# GCP side: service account for the workload
resource "google_service_account" "cross_cloud" {
  account_id   = "cross-cloud-worker"
  display_name = "Cross-cloud worker service account"
  project      = var.gcp_project_id
}

# AWS side: IAM role trusting the GCP service account
resource "aws_iam_role" "gcp_workload" {
  name = "gcp-workload-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = "accounts.google.com"
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "accounts.google.com:sub" = google_service_account.cross_cloud.unique_id
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "gcp_workload_s3" {
  name = "s3-read-access"
  role = aws_iam_role.gcp_workload.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:GetObject", "s3:ListBucket"]
        Resource = [
          aws_s3_bucket.data.arn,
          "${aws_s3_bucket.data.arn}/*"
        ]
      }
    ]
  })
}
```

```hcl
# azure-to-aws.tf — Azure workload accessing AWS resources

# Azure side: user-assigned managed identity
resource "azurerm_user_assigned_identity" "cross_cloud" {
  location            = azurerm_resource_group.main.location
  name                = "cross-cloud-identity"
  resource_group_name = azurerm_resource_group.main.name
}

# AWS side: OIDC provider for Azure AD
resource "aws_iam_openid_connect_provider" "azure_ad" {
  url             = "https://sts.windows.net/${var.azure_tenant_id}/"
  client_id_list  = [var.azure_app_client_id]
  thumbprint_list = [var.azure_ad_thumbprint]
}

# AWS side: role trusting the Azure managed identity
resource "aws_iam_role" "azure_workload" {
  name = "azure-workload-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.azure_ad.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${aws_iam_openid_connect_provider.azure_ad.url}:sub" = azurerm_user_assigned_identity.cross_cloud.principal_id
          }
        }
      }
    ]
  })
}
```

---

## Multi-Cloud State Management

### Separate State per Cloud

The simplest and most commonly recommended pattern is **one state file per cloud per environment**. This provides clear blast radius boundaries and independent lifecycle management:

```
state/
├── aws/
│   ├── production/
│   │   ├── networking.tfstate      # AWS VPCs, subnets, route tables
│   │   ├── compute.tfstate         # AWS EC2, ECS, Lambda
│   │   └── data.tfstate            # AWS RDS, S3, DynamoDB
│   └── staging/
│       ├── networking.tfstate
│       ├── compute.tfstate
│       └── data.tfstate
├── azure/
│   ├── production/
│   │   ├── networking.tfstate      # Azure VNets, subnets
│   │   ├── compute.tfstate         # Azure VMs, AKS
│   │   └── data.tfstate            # Azure SQL, Blob Storage
│   └── staging/
│       ├── networking.tfstate
│       ├── compute.tfstate
│       └── data.tfstate
└── gcp/
    ├── production/
    │   ├── networking.tfstate      # GCP VPCs, subnets
    │   ├── compute.tfstate         # GCP GCE, GKE
    │   └── data.tfstate            # GCP Cloud SQL, GCS
    └── staging/
        ├── networking.tfstate
        ├── compute.tfstate
        └── data.tfstate
```

Each state file uses a backend in its own cloud provider for low-latency access:

```hcl
# aws/production/networking/backend.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "aws/production/networking.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

# azure/production/networking/backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "mycompanytfstate"
    container_name       = "tfstate"
    key                  = "azure/production/networking.tfstate"
  }
}

# gcp/production/networking/backend.tf
terraform {
  backend "gcs" {
    bucket = "mycompany-terraform-state"
    prefix = "gcp/production/networking"
  }
}
```

### Unified State Strategies

Some organizations prefer a **single state backend** for all clouds, trading latency for operational simplicity:

| Strategy | Pros | Cons |
|---|---|---|
| **State per cloud** | Clear blast radius, provider-native backend | Cross-cloud references require data sources or remote state |
| **Single backend** | Centralized management, unified access control | Single point of failure, potential latency for non-primary clouds |
| **Hybrid** | Best of both — cloud-native backends with cross-references | More complex backend configuration |

The hybrid approach uses `terraform_remote_state` data sources to share outputs across cloud boundaries:

```hcl
# gcp/production/compute/main.tf — referencing AWS state

data "terraform_remote_state" "aws_networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "aws/production/networking.tfstate"
    region = "us-east-1"
  }
}

# Use AWS VPN gateway IP to configure the GCP VPN tunnel
resource "google_compute_vpn_tunnel" "to_aws" {
  name          = "gcp-to-aws-tunnel"
  peer_ip       = data.terraform_remote_state.aws_networking.outputs.vpn_gateway_ip
  shared_secret = var.vpn_shared_secret

  target_vpn_gateway = google_compute_vpn_gateway.main.self_link
  region             = "us-central1"

  local_traffic_selector  = ["10.128.0.0/16"]
  remote_traffic_selector = [
    data.terraform_remote_state.aws_networking.outputs.vpc_cidr
  ]
}
```

### State Organization Patterns

| Pattern | Description | Best For |
|---|---|---|
| **Per-cloud, per-environment, per-layer** | `{cloud}/{env}/{layer}.tfstate` | Large multi-cloud deployments |
| **Per-cloud, per-service** | `{cloud}/{service}.tfstate` | Service-oriented teams |
| **Per-environment** | `{env}/{cloud}-{layer}.tfstate` | Environment-centric operations |
| **Monolith per cloud** | `{cloud}/{env}.tfstate` | Small deployments (not recommended at scale) |

---

## Multi-Cloud CI/CD Pipelines

### Deploying to Multiple Clouds from One Pipeline

A multi-cloud CI/CD pipeline deploys infrastructure to each cloud provider in parallel or sequentially, depending on dependencies:

```yaml
# .github/workflows/multi-cloud-deploy.yml

name: Multi-Cloud Infrastructure Deploy

on:
  push:
    branches: [main]
    paths:
      - 'infrastructure/**'
  pull_request:
    branches: [main]
    paths:
      - 'infrastructure/**'

env:
  TF_VERSION: "1.7.0"

jobs:
  plan:
    strategy:
      matrix:
        cloud: [aws, azure, gcp]
        environment: [staging, production]
        exclude:
          - cloud: azure
            environment: staging
    runs-on: ubuntu-latest
    environment: ${{ matrix.cloud }}-${{ matrix.environment }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS Credentials
        if: matrix.cloud == 'aws'
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Configure Azure Credentials
        if: matrix.cloud == 'azure'
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Configure GCP Credentials
        if: matrix.cloud == 'gcp'
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      - name: Terraform Init
        working-directory: infrastructure/${{ matrix.cloud }}/${{ matrix.environment }}
        run: terraform init

      - name: Terraform Plan
        working-directory: infrastructure/${{ matrix.cloud }}/${{ matrix.environment }}
        run: terraform plan -out=tfplan

      - name: Upload Plan
        uses: actions/upload-artifact@v4
        with:
          name: tfplan-${{ matrix.cloud }}-${{ matrix.environment }}
          path: infrastructure/${{ matrix.cloud }}/${{ matrix.environment }}/tfplan

  apply:
    needs: plan
    if: github.ref == 'refs/heads/main'
    strategy:
      matrix:
        cloud: [aws, azure, gcp]
        environment: [staging, production]
        exclude:
          - cloud: azure
            environment: staging
    runs-on: ubuntu-latest
    environment: ${{ matrix.cloud }}-${{ matrix.environment }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Download Plan
        uses: actions/download-artifact@v4
        with:
          name: tfplan-${{ matrix.cloud }}-${{ matrix.environment }}
          path: infrastructure/${{ matrix.cloud }}/${{ matrix.environment }}

      - name: Terraform Apply
        working-directory: infrastructure/${{ matrix.cloud }}/${{ matrix.environment }}
        run: terraform apply tfplan
```

### Provider Credentials Management

Each cloud provider requires its own credential management strategy. The recommended approach is **workload identity federation** (no long-lived secrets):

| Cloud | CI/CD Credential Method | Secret Required |
|---|---|---|
| **AWS** | OIDC with `role-to-assume` | Role ARN only (no access keys) |
| **Azure** | Federated identity with `client-id` | Client ID, Tenant ID (no client secret) |
| **GCP** | Workload Identity Federation | Provider name, SA email (no key file) |

```yaml
# Secure credential configuration example (GitHub Actions)

# AWS — OIDC federation (no long-lived keys)
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::role/github-actions
    aws-region: us-east-1

# Azure — federated identity (no client secret)
- uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

# GCP — workload identity (no service account key)
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/123/locations/global/workloadIdentityPools/github/providers/github
    service_account: terraform@my-project.iam.gserviceaccount.com
```

### Environment-per-Cloud Patterns

For organizations using GitHub Environments (or equivalent in other CI/CD systems), create **one environment per cloud-region combination**:

```
GitHub Environments:
├── aws-staging
│   ├── Required reviewers: none
│   ├── Secrets: AWS_ROLE_ARN
│   └── Protection rules: none
├── aws-production
│   ├── Required reviewers: @platform-team
│   ├── Secrets: AWS_ROLE_ARN
│   └── Protection rules: main branch only
├── azure-production
│   ├── Required reviewers: @platform-team
│   ├── Secrets: AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_SUBSCRIPTION_ID
│   └── Protection rules: main branch only
└── gcp-production
    ├── Required reviewers: @platform-team
    ├── Secrets: GCP_WORKLOAD_IDENTITY, GCP_SERVICE_ACCOUNT
    └── Protection rules: main branch only
```

---

## Cost and Complexity Considerations

### Hidden Costs of Multi-Cloud

Multi-cloud introduces costs that are not immediately visible:

| Cost Category | Description | Impact |
|---|---|---|
| **Data transfer** | Cross-cloud egress charges | $0.01–0.09/GB between clouds, adds up quickly |
| **Tooling** | Additional monitoring, security, and management tools | Each tool may need per-cloud configuration |
| **Training** | Engineers must learn multiple cloud platforms | Slower onboarding, wider but shallower expertise |
| **Licensing** | Some tools charge per cloud or per resource | Enterprise IaC tools, observability platforms |
| **Abstraction maintenance** | Building and maintaining provider-agnostic modules | Ongoing engineering effort that does not ship features |

### Operational Overhead

Every cloud provider has different:

- **API behaviors** — eventual consistency models, rate limits, error codes
- **Resource naming** — constraints, character limits, uniqueness scopes
- **IAM models** — AWS IAM vs Azure RBAC vs GCP IAM have fundamentally different concepts
- **Networking** — VPC/VNet/VPC terminology overlaps but semantics differ
- **Monitoring** — CloudWatch vs Azure Monitor vs Cloud Monitoring require different configurations

The **operational overhead scales non-linearly** — adding a third cloud is proportionally harder than adding a second because the number of cross-cloud interactions grows combinatorially.

### Skill Requirements

Multi-cloud demands a broader skill set from every team member:

```
Single Cloud Team                Multi-Cloud Team
┌─────────────────────┐          ┌─────────────────────────────┐
│                     │          │                             │
│  Deep AWS expertise │          │  AWS + Azure + GCP          │
│                     │          │  (breadth over depth)       │
│  1 IAM model        │          │  3 IAM models               │
│  1 networking model │          │  3 networking models         │
│  1 CLI              │          │  3 CLIs                      │
│  1 monitoring stack │          │  3 monitoring stacks         │
│                     │          │  + cross-cloud orchestration │
└─────────────────────┘          └─────────────────────────────┘
```

### When to Simplify

Consider simplifying your multi-cloud strategy when:

1. **Most workloads are on one cloud** — if 90% of resources are in AWS and 10% are in GCP, the GCP portion may not justify the multi-cloud overhead
2. **Abstractions are unused** — you built provider-agnostic modules but only ever deploy to one provider
3. **The original driver is gone** — the compliance requirement changed, or the acquisition integration is complete
4. **Team velocity is suffering** — multi-cloud complexity is slowing feature delivery beyond acceptable levels
5. **Cost savings never materialized** — the engineering cost of maintaining multi-cloud exceeds any pricing arbitrage

---

## Multi-Cloud vs Hybrid Cloud

### Definition Differences

These terms are often used interchangeably but refer to different strategies:

| Aspect | Multi-Cloud | Hybrid Cloud |
|---|---|---|
| **Definition** | Using multiple public cloud providers | Combining public cloud with on-premises or private cloud |
| **Example** | AWS + GCP + Azure | AWS + on-premises data center |
| **Primary driver** | Best-of-breed, vendor diversification | Data locality, legacy integration, regulatory compliance |
| **Network** | Cloud-to-cloud connectivity | Cloud-to-on-premises connectivity |
| **Identity** | Federated across public clouds | Federated between cloud and enterprise AD |

### Use Cases

**Multi-Cloud Use Cases:**

- Running ML workloads on GCP while serving web traffic from AWS
- Meeting compliance by placing European data in Azure (EU regions) and US data in AWS
- Inheriting GCP infrastructure through an acquisition while primary operations run on AWS

**Hybrid Cloud Use Cases:**

- Extending an on-premises VMware environment into AWS with VMware Cloud
- Running sensitive workloads on-premises while using cloud for burst capacity
- Maintaining a local Kubernetes cluster for development while deploying to cloud for production

### Architecture Differences

```
Multi-Cloud Architecture:
┌──────────┐     ┌──────────┐     ┌──────────┐
│   AWS    │◄───►│  Azure   │◄───►│   GCP    │
│  Public  │     │  Public  │     │  Public  │
│  Cloud   │     │  Cloud   │     │  Cloud   │
└──────────┘     └──────────┘     └──────────┘
    All workloads run in public clouds.

Hybrid Cloud Architecture:
┌──────────┐     ┌────────────────┐
│   AWS    │◄───►│  On-Premises   │
│  Public  │     │  Data Center   │
│  Cloud   │     │  (VMware /     │
│          │     │   OpenStack)   │
└──────────┘     └────────────────┘
    Workloads span public and private infrastructure.

Multi-Cloud + Hybrid Architecture:
┌──────────┐     ┌──────────┐     ┌────────────────┐
│   AWS    │◄───►│   GCP    │◄───►│  On-Premises   │
│  Public  │     │  Public  │     │  Data Center   │
│  Cloud   │     │  Cloud   │     │                │
└──────────┘     └──────────┘     └────────────────┘
    The most complex scenario — multiple public clouds + private.
```

---

## Multi-Cloud Tool Comparison

The three primary tools for multi-cloud IaC are **Terraform**, **Pulumi**, and **Crossplane**. Each has distinct strengths:

| Feature | Terraform | Pulumi | Crossplane |
|---|---|---|---|
| **Language** | HCL (declarative DSL) | TypeScript, Python, Go, C# | YAML (Kubernetes CRDs) |
| **Multi-Cloud Model** | Multiple providers in config | Multiple SDKs in code | Multiple providers as K8s controllers |
| **Abstraction Mechanism** | Modules with conditional logic | Classes, interfaces, generics | XRDs and Compositions |
| **State Management** | State files (S3, GCS, Azure Blob) | State files (Pulumi Cloud, S3) | Kubernetes etcd (no separate state) |
| **Learning Curve** | Low (HCL is simple) | Medium (requires programming) | Medium (requires Kubernetes knowledge) |
| **Type Safety** | Limited (validation blocks) | Full language type systems | OpenAPI schema validation |
| **Testing** | `terraform test`, Terratest | Native unit testing frameworks | Kubernetes-native testing |
| **Drift Detection** | `terraform plan` | `pulumi preview` | Continuous reconciliation |
| **GitOps Native** | No (requires wrapper) | No (requires wrapper) | Yes (Kubernetes-native) |
| **Community Providers** | 4000+ providers | 150+ providers (+ Terraform bridge) | 200+ providers (growing) |
| **Best For** | Teams standardized on HCL | Teams preferring general-purpose languages | Teams already running Kubernetes |

**When to choose each tool:**

- **Terraform** — the default choice for most multi-cloud scenarios; largest provider ecosystem, most community resources, and widest adoption
- **Pulumi** — when your team has strong programming language skills and wants compile-time type safety, unit testing, and code reuse patterns that HCL cannot express
- **Crossplane** — when Kubernetes is already your control plane and you want infrastructure management to follow the same reconciliation model as your application deployments

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
| [05-MODULES-AND-REUSE.md](05-MODULES-AND-REUSE.md) | Modules and Reuse | Module design, versioning, registries |
| [06-TESTING.md](06-TESTING.md) | IaC Testing | Policy as code, plan validation, integration testing |
| [07-SECRETS-AND-VARIABLES.md](07-SECRETS-AND-VARIABLES.md) | Secrets and Variables | Secret management, variable injection, encryption |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Directory structure, naming, CI/CD integration |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial Multi-Cloud IaC documentation |
