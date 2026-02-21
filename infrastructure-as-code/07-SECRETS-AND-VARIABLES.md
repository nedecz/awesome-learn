# Secrets and Variables

A comprehensive guide to managing secrets and variables in infrastructure as code — covering the secrets problem, variable types and hierarchies, environment-specific configuration, sensitive values, secrets manager integration, SOPS encryption, environment variables, dynamic secrets, state file security, variable validation, secrets rotation, and comparison of approaches across Terraform, Pulumi, and CloudFormation.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [The Secrets Problem in IaC](#the-secrets-problem-in-iac)
   - [Why Secrets in Code Are Dangerous](#why-secrets-in-code-are-dangerous)
   - [Attack Vectors](#attack-vectors)
   - [Compliance Requirements](#compliance-requirements)
3. [Variable Types and Hierarchies](#variable-types-and-hierarchies)
   - [Terraform Variables](#terraform-variables)
   - [Variable Precedence](#variable-precedence)
   - [Pulumi Configuration](#pulumi-configuration)
   - [CloudFormation Parameters](#cloudformation-parameters)
4. [Environment-Specific Configuration](#environment-specific-configuration)
   - [Dev, Staging, and Production Variables](#dev-staging-and-production-variables)
   - [Workspace-Based Configuration](#workspace-based-configuration)
   - [Stack-Based Configuration](#stack-based-configuration)
   - [Variable Files per Environment](#variable-files-per-environment)
5. [Sensitive Values in Terraform](#sensitive-values-in-terraform)
   - [The sensitive Attribute](#the-sensitive-attribute)
   - [Marking Outputs Sensitive](#marking-outputs-sensitive)
   - [State File Risks](#state-file-risks)
   - [terraform.tfvars vs Environment Variables](#terraformtfvars-vs-environment-variables)
6. [Secrets Managers Integration](#secrets-managers-integration)
   - [HashiCorp Vault](#hashicorp-vault)
   - [AWS Secrets Manager](#aws-secrets-manager)
   - [Azure Key Vault](#azure-key-vault)
   - [GCP Secret Manager](#gcp-secret-manager)
7. [SOPS (Secrets OPerationS)](#sops-secrets-operations)
   - [Encrypting Files at Rest](#encrypting-files-at-rest)
   - [SOPS with Terraform](#sops-with-terraform)
   - [SOPS with age, PGP, and AWS KMS](#sops-with-age-pgp-and-aws-kms)
8. [Environment Variables for Secrets](#environment-variables-for-secrets)
   - [TF_VAR_ Convention](#tf_var_-convention)
   - [CI/CD Variable Injection](#cicd-variable-injection)
   - [GitHub Actions Secrets](#github-actions-secrets)
   - [Azure DevOps Variable Groups](#azure-devops-variable-groups)
9. [Dynamic Secrets](#dynamic-secrets)
   - [Vault Dynamic Secrets](#vault-dynamic-secrets)
   - [Temporary Credentials](#temporary-credentials)
   - [Lease Management](#lease-management)
   - [Just-in-Time Access](#just-in-time-access)
10. [Secrets in State Files](#secrets-in-state-files)
    - [State Encryption](#state-encryption)
    - [Backend Encryption](#backend-encryption)
    - [Sensitive Data in State](#sensitive-data-in-state)
    - [State Access Controls](#state-access-controls)
11. [Variable Validation](#variable-validation)
    - [Custom Validation Rules](#custom-validation-rules)
    - [Input Constraints](#input-constraints)
    - [Type Checking](#type-checking)
12. [Secrets Rotation](#secrets-rotation)
    - [Automated Rotation Patterns](#automated-rotation-patterns)
    - [Rotation Lambda/Function](#rotation-lambdafunction)
    - [Zero-Downtime Rotation](#zero-downtime-rotation)
13. [Secrets Management Comparison](#secrets-management-comparison)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **managing secrets and variables in infrastructure as code** — the practices, tools, and patterns that enable teams to configure infrastructure safely across environments without exposing sensitive data. Secrets management in IaC spans a wide spectrum, from simple variable files and environment variables for non-sensitive configuration, to dedicated secrets managers, encryption at rest, and dynamic short-lived credentials for production secrets. A well-designed secrets and variables strategy prevents credential leaks, enforces least-privilege access, satisfies compliance requirements, and enables teams to promote infrastructure configurations across environments with confidence.

### Target Audience

- **DevOps Engineers** building and maintaining Terraform, Pulumi, or CloudFormation configurations who need to manage secrets safely across environments
- **Platform Engineers** designing reusable modules and golden paths that accept configuration without hardcoding environment-specific values or credentials
- **Site Reliability Engineers (SREs)** responsible for infrastructure security and who need automated guardrails against secret exposure in state files and logs
- **Security Engineers** enforcing compliance policies around secret storage, rotation, and access across infrastructure definitions
- **Developers** contributing to infrastructure code who need to understand how to pass configuration and secrets without committing them to version control

### Scope

- Why secrets in infrastructure code, state files, and version control are dangerous
- Variable types, hierarchies, and precedence in Terraform, Pulumi, and CloudFormation
- Environment-specific configuration using workspaces, stacks, and variable files
- The sensitive attribute and its limitations in Terraform
- Integrating with HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, and GCP Secret Manager
- SOPS for encrypting secrets files at rest with age, PGP, and cloud KMS
- Environment variable conventions and CI/CD secret injection patterns
- Dynamic secrets, lease management, and just-in-time access with Vault
- State file encryption, backend security, and access controls
- Variable validation rules and input constraints
- Automated secrets rotation patterns for zero-downtime deployments

---

## The Secrets Problem in IaC

### Why Secrets in Code Are Dangerous

Infrastructure as code requires credentials to provision and configure cloud resources — database passwords, API keys, TLS certificates, service account tokens, and encryption keys. When these secrets are stored directly in source code, configuration files, or state files, they become accessible to anyone with repository or backend access. A single leaked database password can lead to a full data breach. A committed cloud provider key can result in thousands of dollars in unauthorized resource creation within minutes.

```
The Secrets Exposure Problem
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Developer writes IaC code
       │
       ▼
  ┌─────────────────────┐
  │  main.tf             │
  │                      │
  │  password = "s3cret" │  ◄── hardcoded secret
  │  api_key  = "abc123" │
  └─────────────────────┘
       │
       ▼
  git commit && git push
       │
       ├──► Repository history (forever)
       ├──► CI/CD build logs
       ├──► Terraform state file (plaintext)
       ├──► Pull request diffs (visible to reviewers)
       └──► Forks and mirrors

  Even after deletion, secrets persist in git history
  and may have been cached, indexed, or scraped.
```

The problem is compounded by the fact that infrastructure code often requires higher-privilege credentials than application code. A Terraform configuration that manages an entire AWS account needs administrative access — meaning a single leaked credential provides full control over the infrastructure.

### Attack Vectors

Secrets in IaC are exposed through multiple attack vectors, each with different risk profiles:

| Attack Vector | Description | Likelihood | Impact |
|---|---|---|---|
| Git history mining | Scanning repository history for committed secrets | High | Critical |
| State file access | Reading plaintext secrets from Terraform state | High | Critical |
| CI/CD log exposure | Secrets printed in build output or plan logs | Medium | High |
| Forked repositories | Public forks retain committed secrets from upstream | Medium | Critical |
| Backup and snapshot access | Secrets in infrastructure snapshots or backups | Medium | High |
| Social engineering | Tricking developers into sharing .tfvars files | Low | High |
| Dependency confusion | Malicious providers or modules exfiltrating variables | Low | Critical |
| State file in local filesystem | terraform.tfstate on developer laptops | High | High |

```
Attack Surface Map
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

                    ┌──────────────┐
                    │  Git Repo    │
                    │  (history)   │
                    └──────┬───────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
    ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐
    │  .tfvars  │   │  .tf files│   │  CI/CD     │
    │  files    │   │  (code)   │   │  logs      │
    └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
          │                │                │
          └────────────────┼────────────────┘
                           │
                    ┌──────▼───────┐
                    │  State File  │
                    │  (plaintext  │
                    │   values)    │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │  Deployed    │
                    │  Resources   │
                    └──────────────┘
```

### Compliance Requirements

Regulatory frameworks mandate strict controls around secret management in infrastructure:

| Framework | Requirement | IaC Implication |
|---|---|---|
| SOC 2 | Secrets must be encrypted at rest and in transit | State files must be encrypted; no plaintext secrets in repos |
| PCI DSS | Cardholder data must be protected; access logged | Database credentials must use secrets managers with audit trails |
| HIPAA | PHI access must be controlled and auditable | Infrastructure secrets for healthcare systems need rotation and logging |
| GDPR | Personal data must be protected with appropriate measures | Encryption keys and access credentials must be managed securely |
| FedRAMP | FIPS 140-2 validated encryption for secrets | Must use approved encryption modules for secret storage |
| ISO 27001 | Information security management controls | Formal secrets management policy and implementation required |

> **Key Principle:** Secrets must never be stored in version control, must be encrypted at rest and in transit, must be rotated regularly, and access must be auditable. Every IaC workflow must be designed with these constraints from the beginning — retrofitting secrets management is significantly harder than building it in.

---

## Variable Types and Hierarchies

### Terraform Variables

Terraform supports several variable types for defining configuration inputs. Variables provide a way to parameterize configurations, making them reusable across environments without modifying the underlying code.

```hcl
# variables.tf — Variable type examples

# String variable with default
variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
}

# Number variable
variable "instance_count" {
  description = "Number of instances to create"
  type        = number
  default     = 2
}

# Boolean variable
variable "enable_monitoring" {
  description = "Whether to enable CloudWatch monitoring"
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
    backup_retention_period = number
    tags           = map(string)
  })
  default = {
    engine         = "postgres"
    engine_version = "15.4"
    instance_class = "db.t3.medium"
    allocated_storage = 100
    multi_az       = false
    backup_retention_period = 7
    tags           = {}
  }
}

# Tuple variable (fixed-length, mixed types)
variable "cidr_config" {
  description = "CIDR block and mask"
  type        = tuple([string, number])
  default     = ["10.0.0.0", 16]
}

# Set variable (unique values)
variable "allowed_regions" {
  description = "Set of allowed AWS regions"
  type        = set(string)
  default     = ["us-east-1", "us-west-2", "eu-west-1"]
}
```

### Variable Precedence

Terraform evaluates variables from multiple sources with a defined precedence order. Understanding this order is critical to avoiding unexpected values in production.

```
Terraform Variable Precedence (highest to lowest)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  1. -var and -var-file CLI flags          ◄── HIGHEST priority
     │
  2. *.auto.tfvars / *.auto.tfvars.json
     │    (alphabetical order)
     │
  3. terraform.tfvars / terraform.tfvars.json
     │
  4. TF_VAR_ environment variables
     │
  5. Variable defaults in .tf files        ◄── LOWEST priority


  Example resolution:
  ┌────────────────────────────────┐
  │  variable "env" {              │
  │    default = "dev"             │  ◄── 5. default
  │  }                             │
  └────────────────────────────────┘
  $ export TF_VAR_env="staging"      ◄── 4. env var
  terraform.tfvars: env = "qa"       ◄── 3. tfvars
  prod.auto.tfvars: env = "preprod"  ◄── 2. auto.tfvars
  $ terraform apply -var="env=prod"  ◄── 1. CLI (wins)

  Result: env = "prod"
```

```bash
# Setting variables at different precedence levels

# 1. CLI flags (highest priority)
terraform apply -var="environment=prod" -var="instance_count=5"
terraform apply -var-file="prod.tfvars"

# 2. Auto-loaded variable files (alphabetical)
# Files named *.auto.tfvars are automatically loaded
# prod.auto.tfvars, staging.auto.tfvars

# 3. Default variable files (auto-loaded)
# terraform.tfvars or terraform.tfvars.json

# 4. Environment variables
export TF_VAR_environment="staging"
export TF_VAR_instance_count="3"

# 5. Default values in variable declarations (lowest priority)
```

### Pulumi Configuration

Pulumi uses a stack-based configuration system where values are stored in YAML files per stack. Secrets are encrypted by default using the stack's encryption provider.

```typescript
// index.ts — Reading Pulumi configuration
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const config = new pulumi.Config();

// Plain configuration values
const environment = config.require("environment");
const instanceCount = config.requireNumber("instanceCount");
const enableMonitoring = config.requireBoolean("enableMonitoring");

// Secret configuration values (automatically encrypted in state)
const dbPassword = config.requireSecret("dbPassword");
const apiKey = config.requireSecret("apiKey");

// Object configuration
const dbConfig = config.requireObject<{
  engine: string;
  instanceClass: string;
  allocatedStorage: number;
}>("databaseConfig");

// Optional with defaults
const region = config.get("region") || "us-east-1";
const retentionDays = config.getNumber("retentionDays") || 7;
```

```yaml
# Pulumi.prod.yaml — Stack configuration file
config:
  app:environment: prod
  app:instanceCount: "5"
  app:enableMonitoring: "true"
  app:region: us-east-1
  # Secrets are encrypted automatically
  app:dbPassword:
    secure: AAABADQw1Z9KxEPBlm4Su...encrypted...
  app:apiKey:
    secure: AAABAHj7K2mNpVxR8tZwQ...encrypted...
  app:databaseConfig:
    engine: postgres
    instanceClass: db.r5.xlarge
    allocatedStorage: 500
```

```bash
# Setting Pulumi configuration values
pulumi config set environment prod
pulumi config set instanceCount 5
pulumi config set --secret dbPassword "my-secret-password"
pulumi config set --secret apiKey "sk-live-abc123"
```

### CloudFormation Parameters

CloudFormation uses parameters to accept input values at stack creation or update time. Parameters support type constraints and the `NoEcho` attribute for sensitive values.

```yaml
# template.yaml — CloudFormation parameters
AWSTemplateFormatVersion: "2010-09-09"
Description: Application infrastructure

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - staging
      - prod
    Description: Deployment environment

  InstanceCount:
    Type: Number
    Default: 2
    MinValue: 1
    MaxValue: 10
    Description: Number of EC2 instances

  DatabasePassword:
    Type: String
    NoEcho: true
    MinLength: 12
    MaxLength: 64
    AllowedPattern: "[a-zA-Z0-9!@#$%^&*]+"
    Description: Database master password (will not be displayed)

  VpcCidr:
    Type: AWS::SSM::Parameter::Value<String>
    Default: /network/vpc-cidr
    Description: VPC CIDR from SSM Parameter Store

  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2
    Description: Latest Amazon Linux 2 AMI from SSM

Resources:
  Database:
    Type: AWS::RDS::DBInstance
    Properties:
      MasterUserPassword: !Ref DatabasePassword
      # ... other properties
```

---

## Environment-Specific Configuration

### Dev, Staging, and Production Variables

Managing different configurations per environment is a fundamental IaC challenge. Each environment may differ in instance sizes, replica counts, feature flags, network ranges, and associated secrets.

```
Environment Configuration Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Shared Code (modules)
       │
       ├──► dev.tfvars ──────► Dev Environment
       │     • small instances
       │     • single AZ
       │     • debug logging
       │
       ├──► staging.tfvars ──► Staging Environment
       │     • medium instances
       │     • multi-AZ
       │     • standard logging
       │
       └──► prod.tfvars ────► Production Environment
             • large instances
             • multi-AZ + multi-region
             • structured logging
             • enhanced monitoring
```

```hcl
# environments/dev.tfvars
environment       = "dev"
instance_type     = "t3.small"
instance_count    = 1
multi_az          = false
enable_monitoring = false
enable_backups    = false
log_level         = "debug"

database_config = {
  instance_class          = "db.t3.micro"
  allocated_storage       = 20
  backup_retention_period = 1
  deletion_protection     = false
}

# environments/staging.tfvars
environment       = "staging"
instance_type     = "t3.medium"
instance_count    = 2
multi_az          = true
enable_monitoring = true
enable_backups    = true
log_level         = "info"

database_config = {
  instance_class          = "db.t3.medium"
  allocated_storage       = 100
  backup_retention_period = 7
  deletion_protection     = true
}

# environments/prod.tfvars
environment       = "prod"
instance_type     = "m5.xlarge"
instance_count    = 4
multi_az          = true
enable_monitoring = true
enable_backups    = true
log_level         = "warn"

database_config = {
  instance_class          = "db.r5.xlarge"
  allocated_storage       = 500
  backup_retention_period = 30
  deletion_protection     = true
}
```

```bash
# Apply with environment-specific variables
terraform apply -var-file="environments/dev.tfvars"
terraform apply -var-file="environments/staging.tfvars"
terraform apply -var-file="environments/prod.tfvars"
```

### Workspace-Based Configuration

Terraform workspaces enable managing multiple environments from a single configuration directory by switching the active workspace. Variables can be selected based on the current workspace.

```hcl
# main.tf — Workspace-based configuration
locals {
  environment = terraform.workspace

  config = {
    dev = {
      instance_type  = "t3.small"
      instance_count = 1
      multi_az       = false
    }
    staging = {
      instance_type  = "t3.medium"
      instance_count = 2
      multi_az       = true
    }
    prod = {
      instance_type  = "m5.xlarge"
      instance_count = 4
      multi_az       = true
    }
  }

  current = local.config[local.environment]
}

resource "aws_instance" "app" {
  count         = local.current.instance_count
  instance_type = local.current.instance_type
  ami           = data.aws_ami.app.id

  tags = {
    Name        = "app-${local.environment}-${count.index}"
    Environment = local.environment
  }
}
```

```bash
# Workspace management
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

terraform workspace select prod
terraform apply
```

### Stack-Based Configuration

Pulumi uses stacks as the primary mechanism for environment separation. Each stack has its own configuration file and state, providing strong isolation between environments.

```typescript
// index.ts — Stack-based Pulumi configuration
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const stack = pulumi.getStack();
const config = new pulumi.Config();

// Environment-specific sizing
const sizing: Record<string, { instanceType: string; count: number }> = {
  dev:     { instanceType: "t3.small",   count: 1 },
  staging: { instanceType: "t3.medium",  count: 2 },
  prod:    { instanceType: "m5.xlarge",  count: 4 },
};

const current = sizing[stack] || sizing["dev"];

for (let i = 0; i < current.count; i++) {
  new aws.ec2.Instance(`app-${i}`, {
    instanceType: current.instanceType,
    ami: "ami-0abcdef1234567890",
    tags: {
      Name: `app-${stack}-${i}`,
      Environment: stack,
    },
  });
}
```

```bash
# Stack management
pulumi stack init dev
pulumi stack init staging
pulumi stack init prod

pulumi stack select prod
pulumi up
```

### Variable Files per Environment

A common directory structure separates variable files by environment while sharing the core Terraform configuration:

```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── backends/
│   ├── dev.hcl
│   ├── staging.hcl
│   └── prod.hcl
├── environments/
│   ├── dev.tfvars
│   ├── staging.tfvars
│   └── prod.tfvars
└── modules/
    ├── networking/
    ├── compute/
    └── database/
```

```hcl
# backends/prod.hcl — Backend configuration per environment
bucket         = "mycompany-terraform-state-prod"
key            = "infrastructure/terraform.tfstate"
region         = "us-east-1"
encrypt        = true
dynamodb_table = "terraform-locks-prod"
```

```bash
# Initialize with environment-specific backend
terraform init -backend-config="backends/prod.hcl"
terraform apply -var-file="environments/prod.tfvars"
```

---

## Sensitive Values in Terraform

### The sensitive Attribute

Terraform 0.14 introduced the `sensitive` attribute for variables and outputs. When a variable is marked sensitive, Terraform redacts its value from CLI output and plan displays. However, the value is still stored in plaintext in the state file.

```hcl
# variables.tf — Sensitive variable declarations
variable "db_password" {
  description = "Database master password"
  type        = string
  sensitive   = true
}

variable "api_key" {
  description = "External API key"
  type        = string
  sensitive   = true
}

variable "tls_private_key" {
  description = "TLS private key for the load balancer"
  type        = string
  sensitive   = true
}

# main.tf — Using sensitive variables
resource "aws_db_instance" "main" {
  identifier     = "app-db-${var.environment}"
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.t3.medium"

  username = "admin"
  password = var.db_password  # sensitive — redacted in plan output

  skip_final_snapshot = var.environment != "prod"
}

resource "aws_ssm_parameter" "api_key" {
  name  = "/${var.environment}/app/api-key"
  type  = "SecureString"
  value = var.api_key  # sensitive — redacted in plan output
}
```

```
Plan Output with Sensitive Values
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  # aws_db_instance.main will be created
  + resource "aws_db_instance" "main" {
      + identifier     = "app-db-prod"
      + engine         = "postgres"
      + engine_version = "15.4"
      + instance_class = "db.t3.medium"
      + username       = "admin"
      + password       = (sensitive value)   ◄── redacted
    }
```

> **Warning:** The `sensitive` attribute only controls CLI output visibility. The actual secret value is still stored in the Terraform state file in plaintext. You must protect the state file with encryption and access controls.

### Marking Outputs Sensitive

Outputs that contain or derive from sensitive values must also be marked sensitive. Terraform will error if you attempt to output a sensitive value without the `sensitive = true` attribute.

```hcl
# outputs.tf — Sensitive outputs
output "db_connection_string" {
  description = "Database connection string"
  value       = "postgres://admin:${var.db_password}@${aws_db_instance.main.endpoint}/appdb"
  sensitive   = true
}

output "db_endpoint" {
  description = "Database endpoint (non-sensitive)"
  value       = aws_db_instance.main.endpoint
}

output "api_key_arn" {
  description = "ARN of the API key parameter"
  value       = aws_ssm_parameter.api_key.arn
  # ARN is not sensitive — no need to mark it
}
```

### State File Risks

Even with the `sensitive` attribute, Terraform state files contain the actual values of all managed resources — including passwords, keys, and certificates. The state file is the single most sensitive artifact in any Terraform workflow.

```
State File Sensitivity
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  terraform.tfstate (JSON)
  ┌──────────────────────────────────────────┐
  │ {                                        │
  │   "resources": [{                        │
  │     "type": "aws_db_instance",           │
  │     "instances": [{                      │
  │       "attributes": {                    │
  │         "password": "actual-password"  ◄─┤── PLAINTEXT!
  │         "endpoint": "db.example.com"     │
  │       }                                  │
  │     }]                                   │
  │   }]                                     │
  │ }                                        │
  └──────────────────────────────────────────┘

  Even with sensitive = true, the state file
  contains the real value. Protect the state!
```

### terraform.tfvars vs Environment Variables

Both `terraform.tfvars` and environment variables can supply secrets, but they have different security profiles:

| Method | Persisted to Disk | In Git History | Visible in Process List | Best For |
|---|---|---|---|---|
| terraform.tfvars | Yes | Risk if committed | No | Non-sensitive defaults |
| *.auto.tfvars | Yes | Risk if committed | No | Non-sensitive overrides |
| TF_VAR_ env vars | No (memory only) | No | `ps -e` can show | CI/CD pipelines |
| -var CLI flag | No | No | Yes (`ps -e`) | One-off overrides |
| Secrets manager | No | No | No | Production secrets |

```bash
# Using environment variables for secrets (recommended for CI/CD)
export TF_VAR_db_password="$(vault kv get -field=password secret/database)"
export TF_VAR_api_key="$(vault kv get -field=key secret/api)"

terraform apply -var-file="environments/prod.tfvars"
# prod.tfvars contains non-sensitive values only
```

---

## Secrets Managers Integration

### HashiCorp Vault

HashiCorp Vault is a dedicated secrets management platform that provides dynamic secrets, encryption as a service, and fine-grained access control. Terraform integrates with Vault through the `vault` provider and data sources.

```hcl
# providers.tf — Vault provider configuration
provider "vault" {
  address = "https://vault.example.com:8200"
  # Authentication via VAULT_TOKEN env var or other auth methods
}

# main.tf — Reading secrets from Vault KV store
data "vault_kv_secret_v2" "database" {
  mount = "secret"
  name  = "database/prod"
}

data "vault_kv_secret_v2" "api_keys" {
  mount = "secret"
  name  = "api-keys/prod"
}

resource "aws_db_instance" "main" {
  identifier     = "app-db-prod"
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.r5.xlarge"

  username = data.vault_kv_secret_v2.database.data["username"]
  password = data.vault_kv_secret_v2.database.data["password"]
}

resource "aws_ssm_parameter" "api_key" {
  name  = "/prod/app/external-api-key"
  type  = "SecureString"
  value = data.vault_kv_secret_v2.api_keys.data["external_api_key"]
}
```

```typescript
// Pulumi with Vault
import * as pulumi from "@pulumi/pulumi";
import * as vault from "@pulumi/vault";
import * as aws from "@pulumi/aws";

const dbSecret = vault.kv.getSecretV2({
  mount: "secret",
  name: "database/prod",
});

const db = new aws.rds.Instance("main", {
  identifier:    "app-db-prod",
  engine:        "postgres",
  engineVersion: "15.4",
  instanceClass: "db.r5.xlarge",
  username:      dbSecret.then(s => s.data["username"]),
  password:      dbSecret.then(s => s.data["password"]),
});
```

### AWS Secrets Manager

AWS Secrets Manager provides native secret storage with automatic rotation, fine-grained IAM policies, and cross-account access. Terraform reads secrets using the `aws_secretsmanager_secret_version` data source.

```hcl
# main.tf — Reading from AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = "prod/database/credentials"
}

locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.db_credentials.secret_string)
}

resource "aws_db_instance" "main" {
  identifier     = "app-db-prod"
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.r5.xlarge"

  username = local.db_creds["username"]
  password = local.db_creds["password"]
}

# Creating a secret with Terraform
resource "aws_secretsmanager_secret" "app_api_key" {
  name                    = "prod/app/api-key"
  description             = "External API key for the application"
  recovery_window_in_days = 7
}

resource "aws_secretsmanager_secret_version" "app_api_key" {
  secret_id     = aws_secretsmanager_secret.app_api_key.id
  secret_string = jsonencode({
    api_key    = var.api_key
    created_by = "terraform"
  })
}
```

### Azure Key Vault

Azure Key Vault provides centralized secret, key, and certificate management with RBAC integration and audit logging.

```hcl
# main.tf — Reading from Azure Key Vault
data "azurerm_key_vault" "main" {
  name                = "myapp-kv-prod"
  resource_group_name = "rg-shared-prod"
}

data "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  key_vault_id = data.azurerm_key_vault.main.id
}

data "azurerm_key_vault_secret" "api_key" {
  name         = "external-api-key"
  key_vault_id = data.azurerm_key_vault.main.id
}

resource "azurerm_mssql_server" "main" {
  name                         = "app-sql-prod"
  resource_group_name          = azurerm_resource_group.main.name
  location                     = azurerm_resource_group.main.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = data.azurerm_key_vault_secret.db_password.value
}
```

```typescript
// Pulumi with Azure Key Vault
import * as pulumi from "@pulumi/pulumi";
import * as azure from "@pulumi/azure-native";
import * as keyvault from "@pulumi/azure-native/keyvault";

const vault = keyvault.getVaultOutput({
  vaultName: "myapp-kv-prod",
  resourceGroupName: "rg-shared-prod",
});

const dbPassword = keyvault.getSecretOutput({
  vaultName: "myapp-kv-prod",
  resourceGroupName: "rg-shared-prod",
  secretName: "db-password",
});

const sqlServer = new azure.sql.Server("main", {
  serverName: "app-sql-prod",
  resourceGroupName: "rg-app-prod",
  location: "eastus",
  version: "12.0",
  administratorLogin: "sqladmin",
  administratorLoginPassword: dbPassword.apply(s => s.value!),
});
```

### GCP Secret Manager

Google Cloud Secret Manager provides a centralized and secure API for managing secrets with IAM-based access control and audit logging.

```hcl
# main.tf — Reading from GCP Secret Manager
data "google_secret_manager_secret_version" "db_password" {
  secret  = "db-password"
  project = "my-project-prod"
}

data "google_secret_manager_secret_version" "api_key" {
  secret  = "external-api-key"
  project = "my-project-prod"
}

resource "google_sql_database_instance" "main" {
  name             = "app-db-prod"
  database_version = "POSTGRES_15"
  region           = "us-central1"

  settings {
    tier = "db-custom-4-16384"
  }
}

resource "google_sql_user" "admin" {
  name     = "admin"
  instance = google_sql_database_instance.main.name
  password = data.google_secret_manager_secret_version.db_password.secret_data
}

# Creating a secret
resource "google_secret_manager_secret" "app_secret" {
  secret_id = "app-config"
  project   = "my-project-prod"

  replication {
    auto {}
  }
}

resource "google_secret_manager_secret_version" "app_secret" {
  secret      = google_secret_manager_secret.app_secret.id
  secret_data = jsonencode({
    api_key = var.api_key
    region  = "us-central1"
  })
}
```

---

## SOPS (Secrets OPerationS)

### Encrypting Files at Rest

SOPS (Secrets OPerationS) is a tool for encrypting and decrypting files while preserving their structure. Unlike full-file encryption, SOPS encrypts only the values in structured files (YAML, JSON, ENV, INI), leaving keys in plaintext. This allows meaningful diffs, code reviews, and version control of encrypted secrets files.

```
SOPS Encryption Model
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Plaintext file:              Encrypted file:
  ┌─────────────────────┐      ┌──────────────────────────────┐
  │ db_password: s3cret  │ ──► │ db_password: ENC[AES256...   │
  │ api_key: abc123      │      │ api_key: ENC[AES256...       │
  │ region: us-east-1    │      │ region: us-east-1            │
  └─────────────────────┘      │ sops:                        │
                                │   kms: ...                   │
  Keys are visible ────────►   │   age: ...                   │
  Values are encrypted ────►   └──────────────────────────────┘

  Benefits:
  • Meaningful git diffs (keys visible, values opaque)
  • Code review friendly
  • Structure preserved
  • Multiple encryption backends
```

```yaml
# secrets.enc.yaml — SOPS-encrypted file (committed to git)
db_password: ENC[AES256_GCM,data:8fK2...truncated...,type:str]
api_key: ENC[AES256_GCM,data:p7xM...truncated...,type:str]
tls_cert: ENC[AES256_GCM,data:Qw9R...truncated...,type:str]
sops:
  kms:
    - arn: arn:aws:kms:us-east-1:123456789:key/abc-def-123
      created_at: "2025-01-15T10:00:00Z"
      enc: AQICAHh...truncated...
  age:
    - recipient: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
      enc: |
        -----BEGIN AGE ENCRYPTED FILE-----
        YWdlLWVuY3J5cHRpb24...truncated...
        -----END AGE ENCRYPTED FILE-----
  lastmodified: "2025-01-15T10:00:00Z"
  version: 3.8.1
```

```bash
# SOPS basic operations
# Encrypt a file
sops --encrypt --age age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p \
  secrets.yaml > secrets.enc.yaml

# Decrypt a file
sops --decrypt secrets.enc.yaml > secrets.yaml

# Edit in-place (decrypts, opens editor, re-encrypts on save)
sops secrets.enc.yaml

# Rotate encryption keys
sops --rotate --in-place secrets.enc.yaml
```

### SOPS with Terraform

The `sops` Terraform provider decrypts SOPS-encrypted files at plan/apply time, making secrets available as data sources without storing plaintext files in the repository.

```hcl
# providers.tf
terraform {
  required_providers {
    sops = {
      source  = "carlpett/sops"
      version = "~> 1.0"
    }
  }
}

# main.tf — Using SOPS-encrypted files
data "sops_file" "secrets" {
  source_file = "secrets.enc.yaml"
}

resource "aws_db_instance" "main" {
  identifier     = "app-db-${var.environment}"
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = var.db_instance_class

  username = "admin"
  password = data.sops_file.secrets.data["db_password"]
}

resource "aws_ssm_parameter" "api_key" {
  name  = "/${var.environment}/app/api-key"
  type  = "SecureString"
  value = data.sops_file.secrets.data["api_key"]
}
```

### SOPS with age, PGP, and AWS KMS

SOPS supports multiple encryption backends. Each backend has different trade-offs around key management, team workflows, and cloud integration.

| Backend | Key Management | Best For | Team Workflow |
|---|---|---|---|
| age | File-based key pairs | Simple teams, local dev | Share public key; protect private key |
| PGP/GPG | Keyring-based | Teams with existing GPG infrastructure | Each member has own key pair |
| AWS KMS | IAM-managed | AWS-native teams | IAM policies control access |
| GCP KMS | IAM-managed | GCP-native teams | IAM policies control access |
| Azure Key Vault | RBAC-managed | Azure-native teams | RBAC policies control access |

```yaml
# .sops.yaml — SOPS configuration (repository root)
creation_rules:
  # Production secrets — encrypted with AWS KMS and age
  - path_regex: environments/prod/.*\.enc\.yaml$
    kms: "arn:aws:kms:us-east-1:123456789:key/prod-key-id"
    age: "age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p"

  # Staging secrets — encrypted with age only
  - path_regex: environments/staging/.*\.enc\.yaml$
    age: "age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p"

  # Dev secrets — encrypted with age only
  - path_regex: environments/dev/.*\.enc\.yaml$
    age: "age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p"
```

```bash
# age key generation
age-keygen -o keys.txt
# Public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p

# Set the age private key for decryption
export SOPS_AGE_KEY_FILE="$HOME/.sops/keys.txt"

# Encrypt with AWS KMS
sops --encrypt --kms "arn:aws:kms:us-east-1:123456789:key/abc-123" \
  secrets.yaml > secrets.enc.yaml

# Encrypt with multiple backends (any one can decrypt)
sops --encrypt \
  --kms "arn:aws:kms:us-east-1:123456789:key/abc-123" \
  --age "age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p" \
  secrets.yaml > secrets.enc.yaml
```

---

## Environment Variables for Secrets

### TF_VAR_ Convention

Terraform automatically reads environment variables prefixed with `TF_VAR_` and uses them as input variable values. This is the most common method for injecting secrets in CI/CD pipelines because it avoids writing secrets to disk.

```bash
# Setting secrets via environment variables
export TF_VAR_db_password="super-secret-password"
export TF_VAR_api_key="sk-live-abc123def456"
export TF_VAR_tls_private_key="$(cat /path/to/key.pem)"

# Terraform automatically maps these to variables:
#   TF_VAR_db_password   → var.db_password
#   TF_VAR_api_key       → var.api_key
#   TF_VAR_tls_private_key → var.tls_private_key

terraform plan
terraform apply
```

```hcl
# variables.tf — Variables populated by TF_VAR_ env vars
variable "db_password" {
  description = "Database password (set via TF_VAR_db_password)"
  type        = string
  sensitive   = true
  # No default — must be provided via env var or other method
}

variable "api_key" {
  description = "API key (set via TF_VAR_api_key)"
  type        = string
  sensitive   = true
}
```

### CI/CD Variable Injection

CI/CD platforms provide secure variable storage that injects values into pipeline steps as environment variables. These values are masked in logs and encrypted at rest.

```
CI/CD Secret Injection Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────────┐     ┌─────────────────┐
  │  CI/CD Platform │     │  Secrets Store  │
  │  (orchestrator) │────►│  (encrypted)    │
  └────────┬────────┘     └─────────────────┘
           │
           │  inject as env vars
           ▼
  ┌─────────────────┐
  │  Pipeline Step  │
  │                 │
  │  TF_VAR_db_pass │  ◄── from secrets store
  │  TF_VAR_api_key │  ◄── from secrets store
  │                 │
  │  terraform apply│
  └─────────────────┘
           │
           │  masked in logs
           ▼
  ┌─────────────────┐
  │  Build Output   │
  │                 │
  │  password = *** │  ◄── redacted
  └─────────────────┘
```

### GitHub Actions Secrets

GitHub Actions provides encrypted secrets at the repository, environment, and organization level. Secrets are injected as environment variables and automatically masked in workflow logs.

```yaml
# .github/workflows/terraform.yml
name: Terraform Apply

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  apply:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Terraform Init
        run: terraform init -backend-config="backends/prod.hcl"

      - name: Terraform Apply
        run: terraform apply -auto-approve -var-file="environments/prod.tfvars"
        env:
          TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}
          TF_VAR_api_key: ${{ secrets.API_KEY }}
          TF_VAR_tls_private_key: ${{ secrets.TLS_PRIVATE_KEY }}
```

```yaml
# .github/workflows/pulumi.yml
name: Pulumi Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - run: npm ci

      - uses: pulumi/actions@v5
        with:
          command: up
          stack-name: prod
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### Azure DevOps Variable Groups

Azure DevOps provides variable groups that can be linked to Azure Key Vault, allowing pipelines to consume secrets without duplicating them across systems.

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

variables:
  - group: terraform-secrets-prod  # linked to Azure Key Vault

stages:
  - stage: Deploy
    jobs:
      - job: TerraformApply
        steps:
          - task: TerraformInstaller@1
            inputs:
              terraformVersion: latest

          - task: TerraformTaskV4@4
            displayName: Terraform Init
            inputs:
              provider: azurerm
              command: init
              backendServiceArm: azure-service-connection
              backendAzureRmResourceGroupName: rg-terraform-state
              backendAzureRmStorageAccountName: tfstateprod
              backendAzureRmContainerName: tfstate
              backendAzureRmKey: prod.terraform.tfstate

          - task: TerraformTaskV4@4
            displayName: Terraform Apply
            inputs:
              provider: azurerm
              command: apply
              commandOptions: -var-file="environments/prod.tfvars"
              environmentServiceNameAzureRM: azure-service-connection
            env:
              TF_VAR_db_password: $(db-password)
              TF_VAR_api_key: $(api-key)
```

---

## Dynamic Secrets

### Vault Dynamic Secrets

Dynamic secrets are generated on-demand by Vault and automatically revoked after a configurable TTL. Unlike static secrets that are shared and rotated, dynamic secrets are unique per request, short-lived, and provide a full audit trail of who accessed what and when.

```
Dynamic Secrets Lifecycle
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Terraform requests credentials
       │
       ▼
  ┌──────────┐    generate     ┌──────────────┐
  │  Vault   │ ──────────────► │  Database     │
  │  Server  │    unique       │  (creates     │
  │          │    credentials  │   temp user)  │
  └──────────┘                 └──────────────┘
       │
       │  return credentials
       │  + lease ID + TTL
       ▼
  ┌──────────────┐
  │  Terraform   │
  │  provisions  │
  │  resources   │
  └──────────────┘
       │
       │  after TTL expires
       ▼
  ┌──────────┐    revoke       ┌──────────────┐
  │  Vault   │ ──────────────► │  Database     │
  │  Server  │                 │  (drops       │
  │          │                 │   temp user)  │
  └──────────┘                 └──────────────┘
```

```hcl
# Vault dynamic database credentials with Terraform
data "vault_generic_secret" "db_creds" {
  path = "database/creds/app-role"
}

# The returned credentials are unique and short-lived
resource "aws_db_instance" "main" {
  identifier     = "app-db-prod"
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.r5.xlarge"

  # These credentials are temporary and auto-revoked
  username = data.vault_generic_secret.db_creds.data["username"]
  password = data.vault_generic_secret.db_creds.data["password"]
}

# Vault AWS dynamic credentials
data "vault_aws_access_credentials" "deploy" {
  backend = "aws"
  role    = "deploy-role"
  type    = "sts"
}

provider "aws" {
  alias      = "dynamic"
  access_key = data.vault_aws_access_credentials.deploy.access_key
  secret_key = data.vault_aws_access_credentials.deploy.secret_key
  token      = data.vault_aws_access_credentials.deploy.security_token
  region     = "us-east-1"
}
```

### Temporary Credentials

Cloud providers offer mechanisms for generating temporary credentials that eliminate the need for long-lived access keys. These integrate directly with IaC tools.

```hcl
# AWS STS Assume Role for temporary credentials
provider "aws" {
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/TerraformDeployRole"
    session_name = "terraform-deploy"
    duration     = "1h"
  }
}

# OIDC federation — no static credentials at all
# Used with GitHub Actions, GitLab CI, etc.
provider "aws" {
  region = "us-east-1"

  assume_role_with_web_identity {
    role_arn                = "arn:aws:iam::123456789012:role/GitHubActionsRole"
    session_name            = "github-actions"
    web_identity_token_file = "/var/run/secrets/token"
  }
}
```

```yaml
# GitHub Actions with OIDC — zero static secrets
name: Deploy with OIDC

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1
          # No access key or secret key needed!

      - run: terraform apply -auto-approve
```

### Lease Management

Vault dynamic secrets include a lease — a contract that specifies the TTL and whether the credential can be renewed. Lease management is critical for long-running Terraform operations.

```hcl
# Vault secrets engine configuration for lease management
resource "vault_database_secret_backend_role" "app" {
  backend = vault_mount.database.path
  name    = "app-role"
  db_name = vault_database_secret_backend_connection.postgres.name

  creation_statements = [
    "CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';",
    "GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";",
  ]

  revocation_statements = [
    "DROP ROLE IF EXISTS \"{{name}}\";",
  ]

  default_ttl = 3600   # 1 hour
  max_ttl     = 86400  # 24 hours
}
```

### Just-in-Time Access

Just-in-time (JIT) access combines dynamic secrets with approval workflows. Access is granted only when needed, for the minimum time required, and automatically revoked.

```
Just-in-Time Access Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Developer requests access
       │
       ▼
  ┌──────────────┐
  │  Approval    │──► Manager approves (Slack, PagerDuty)
  │  Workflow    │
  └──────┬───────┘
         │  approved
         ▼
  ┌──────────────┐
  │  Vault       │──► Generate short-lived credentials
  │  (dynamic)   │    TTL: 30 minutes
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  Terraform   │──► Apply changes
  │  Apply       │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │  Auto-Revoke │──► Credentials destroyed after TTL
  └──────────────┘
```

---

## Secrets in State Files

### State Encryption

Terraform state files contain the complete state of managed infrastructure, including any sensitive values passed to resources. Encrypting the state file at rest is a critical security requirement.

```hcl
# S3 backend with encryption
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true                        # SSE-S3 encryption
    kms_key_id     = "arn:aws:kms:us-east-1:123456789:key/abc-123"  # SSE-KMS
    dynamodb_table = "terraform-locks"
  }
}

# Azure backend with encryption
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "tfstateprod"          # encrypted by default
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}

# GCS backend with encryption
terraform {
  backend "gcs" {
    bucket  = "mycompany-terraform-state"
    prefix  = "prod"
    # GCS encrypts at rest by default
    # Optional: use customer-managed encryption key (CMEK)
  }
}
```

### Backend Encryption

Each backend has different encryption capabilities and defaults:

| Backend | Encryption at Rest | Encryption in Transit | Customer-Managed Keys | Audit Logging |
|---|---|---|---|---|
| S3 | SSE-S3 / SSE-KMS | HTTPS enforced | Yes (KMS) | CloudTrail |
| Azure Blob | AES-256 default | HTTPS enforced | Yes (Key Vault) | Azure Monitor |
| GCS | AES-256 default | HTTPS enforced | Yes (Cloud KMS) | Cloud Audit Logs |
| Terraform Cloud | AES-256 | TLS 1.2+ | Sentinel policies | Audit logs |
| Consul | At rest optional | mTLS | Yes | Audit logs |
| Local file | None | N/A | N/A | None |

> **Warning:** Never use a local backend for production infrastructure. Local state files are unencrypted, have no access controls, no locking, and no audit trail. Always use an encrypted remote backend with access controls.

### Sensitive Data in State

Even when using the `sensitive` attribute, Terraform stores the actual values in state. Understanding what ends up in state helps you protect it appropriately.

```hcl
# These resource attributes contain sensitive data in state:
resource "aws_db_instance" "main" {
  password = var.db_password  # stored in state as plaintext
}

resource "aws_secretsmanager_secret_version" "api" {
  secret_string = var.api_key  # stored in state as plaintext
}

resource "tls_private_key" "cert" {
  algorithm = "RSA"
  rsa_bits  = 4096
  # private_key_pem — stored in state as plaintext
}

# Mitigation: read secrets at deploy time, not through Terraform
# Option 1: Use data sources (still in state, but not managed)
# Option 2: Use external secret injection (e.g., Kubernetes external-secrets)
# Option 3: Use Vault dynamic secrets with short TTL
```

### State Access Controls

Restrict who and what can read or modify the state file using IAM policies, RBAC, and backend-specific access controls.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowTerraformStateAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::mycompany-terraform-state/prod/*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalTag/Team": "platform"
        }
      }
    },
    {
      "Sid": "DenyUnencryptedUploads",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::mycompany-terraform-state/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
```

```hcl
# S3 bucket policy enforcing encryption and access controls
resource "aws_s3_bucket_policy" "state" {
  bucket = aws_s3_bucket.terraform_state.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyUnencryptedState"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:PutObject"
        Resource  = "${aws_s3_bucket.terraform_state.arn}/*"
        Condition = {
          StringNotEquals = {
            "s3:x-amz-server-side-encryption" = "aws:kms"
          }
        }
      },
      {
        Sid       = "DenyHTTP"
        Effect    = "Deny"
        Principal = "*"
        Action    = "s3:*"
        Resource = [
          aws_s3_bucket.terraform_state.arn,
          "${aws_s3_bucket.terraform_state.arn}/*"
        ]
        Condition = {
          Bool = {
            "aws:SecureTransport" = "false"
          }
        }
      }
    ]
  })
}
```

---

## Variable Validation

### Custom Validation Rules

Terraform 0.13 introduced `validation` blocks that enforce constraints on input variables at plan time, preventing invalid configurations from being applied.

```hcl
# variables.tf — Custom validation rules

variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string

  validation {
    condition     = can(regex("^(t3|m5|r5|c5)\\.", var.instance_type))
    error_message = "Instance type must be from an approved family (t3, m5, r5, c5)."
  }
}

variable "db_password" {
  description = "Database master password"
  type        = string
  sensitive   = true

  validation {
    condition     = length(var.db_password) >= 16
    error_message = "Database password must be at least 16 characters."
  }

  validation {
    condition     = can(regex("[A-Z]", var.db_password)) && can(regex("[a-z]", var.db_password)) && can(regex("[0-9]", var.db_password))
    error_message = "Database password must contain uppercase, lowercase, and numeric characters."
  }
}

variable "cidr_block" {
  description = "VPC CIDR block"
  type        = string

  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block (e.g., 10.0.0.0/16)."
  }

  validation {
    condition     = tonumber(split("/", var.cidr_block)[1]) >= 16 && tonumber(split("/", var.cidr_block)[1]) <= 24
    error_message = "CIDR block mask must be between /16 and /24."
  }
}
```

### Input Constraints

Variable validation can enforce business rules and organizational policies, catching configuration errors before they reach the cloud provider API.

```hcl
# Organizational policy constraints
variable "project_name" {
  description = "Project name for resource naming"
  type        = string

  validation {
    condition     = length(var.project_name) >= 3 && length(var.project_name) <= 20
    error_message = "Project name must be between 3 and 20 characters."
  }

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]*$", var.project_name))
    error_message = "Project name must start with a letter and contain only lowercase alphanumeric characters and hyphens."
  }
}

variable "backup_retention_days" {
  description = "Number of days to retain backups"
  type        = number

  validation {
    condition     = var.backup_retention_days >= 7
    error_message = "Backup retention must be at least 7 days per organizational policy."
  }

  validation {
    condition     = var.backup_retention_days <= 365
    error_message = "Backup retention cannot exceed 365 days."
  }
}

variable "allowed_ip_ranges" {
  description = "IP ranges allowed to access the service"
  type        = list(string)

  validation {
    condition     = length(var.allowed_ip_ranges) > 0
    error_message = "At least one IP range must be specified."
  }

  validation {
    condition     = alltrue([for cidr in var.allowed_ip_ranges : can(cidrhost(cidr, 0))])
    error_message = "All entries must be valid CIDR blocks."
  }

  validation {
    condition     = !contains(var.allowed_ip_ranges, "0.0.0.0/0")
    error_message = "Open access (0.0.0.0/0) is not allowed per security policy."
  }
}
```

### Type Checking

Complex type constraints prevent structural errors in variable inputs, especially for object and map types passed to modules.

```hcl
# Complex type constraints
variable "notification_config" {
  description = "Notification configuration"
  type = object({
    enabled    = bool
    channels   = list(string)
    thresholds = map(number)
    recipients = list(object({
      name  = string
      email = string
      role  = string
    }))
  })

  validation {
    condition     = !var.notification_config.enabled || length(var.notification_config.channels) > 0
    error_message = "If notifications are enabled, at least one channel must be specified."
  }

  validation {
    condition = alltrue([
      for r in var.notification_config.recipients :
      can(regex("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$", r.email))
    ])
    error_message = "All recipient email addresses must be valid."
  }
}
```

---

## Secrets Rotation

### Automated Rotation Patterns

Secrets rotation replaces credentials on a schedule or trigger without downtime. The rotation pattern depends on whether the consumer can handle credential changes gracefully.

```
Secrets Rotation Pattern
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Two-User Rotation (AWS Secrets Manager default)
  ┌──────────────────────────────────────────────┐
  │                                              │
  │  Step 1: Create new credential (user-B)      │
  │          ┌─────┐    ┌─────┐                  │
  │          │ A ✓ │    │ B   │                   │
  │          └─────┘    └─────┘                   │
  │                                              │
  │  Step 2: Set new credential in secret        │
  │          ┌─────┐    ┌─────┐                  │
  │          │ A ✓ │    │ B ✓ │ ◄── AWSCURRENT   │
  │          └─────┘    └─────┘                   │
  │                                              │
  │  Step 3: Test new credential                 │
  │          ┌─────┐    ┌─────┐                  │
  │          │ A   │    │ B ✓ │ ◄── verified     │
  │          └─────┘    └─────┘                   │
  │                                              │
  │  Step 4: Remove old credential               │
  │          ┌─────┐    ┌─────┐                  │
  │          │ A ✗ │    │ B ✓ │                   │
  │          └─────┘    └─────┘                   │
  └──────────────────────────────────────────────┘
```

### Rotation Lambda/Function

AWS Secrets Manager supports automatic rotation via Lambda functions. Terraform can provision both the secret and its rotation configuration.

```hcl
# Automated secret rotation with AWS Secrets Manager
resource "aws_secretsmanager_secret" "db_password" {
  name        = "prod/database/password"
  description = "RDS master password with automatic rotation"

  tags = {
    Environment = "prod"
    ManagedBy   = "terraform"
  }
}

resource "aws_secretsmanager_secret_rotation" "db_password" {
  secret_id           = aws_secretsmanager_secret.db_password.id
  rotation_lambda_arn = aws_lambda_function.secret_rotation.arn

  rotation_rules {
    automatically_after_days = 30
  }
}

resource "aws_lambda_function" "secret_rotation" {
  function_name = "secret-rotation-prod"
  handler       = "rotation.handler"
  runtime       = "python3.12"
  timeout       = 60

  filename         = data.archive_file.rotation_lambda.output_path
  source_code_hash = data.archive_file.rotation_lambda.output_base64sha256

  role = aws_iam_role.rotation_lambda.arn

  environment {
    variables = {
      SECRETS_MANAGER_ENDPOINT = "https://secretsmanager.us-east-1.amazonaws.com"
    }
  }
}

resource "aws_lambda_permission" "secrets_manager" {
  statement_id  = "AllowSecretsManagerInvocation"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.secret_rotation.function_name
  principal     = "secretsmanager.amazonaws.com"
}
```

```python
# rotation.py — Secrets Manager rotation Lambda
import json
import boto3

secrets_client = boto3.client("secretsmanager")


def handler(event, context):
    """Rotate a secret in AWS Secrets Manager."""
    secret_arn = event["SecretId"]
    token = event["ClientRequestToken"]
    step = event["Step"]

    if step == "createSecret":
        create_secret(secret_arn, token)
    elif step == "setSecret":
        set_secret(secret_arn, token)
    elif step == "testSecret":
        test_secret(secret_arn, token)
    elif step == "finishSecret":
        finish_secret(secret_arn, token)
    else:
        raise ValueError(f"Unknown rotation step: {step}")


def create_secret(arn, token):
    """Generate a new secret value."""
    import secrets
    import string

    alphabet = string.ascii_letters + string.digits + "!@#$%^&*"
    new_password = "".join(secrets.choice(alphabet) for _ in range(32))

    secrets_client.put_secret_value(
        SecretId=arn,
        ClientRequestToken=token,
        SecretString=json.dumps({"password": new_password}),
        VersionStages=["AWSPENDING"],
    )


def set_secret(arn, token):
    """Set the new credential in the target service."""
    pending = secrets_client.get_secret_value(
        SecretId=arn, VersionId=token, VersionStage="AWSPENDING"
    )
    new_password = json.loads(pending["SecretString"])["password"]

    # Update the database password using admin credentials
    # ... database-specific logic here ...


def test_secret(arn, token):
    """Verify the new credential works."""
    pending = secrets_client.get_secret_value(
        SecretId=arn, VersionId=token, VersionStage="AWSPENDING"
    )
    new_password = json.loads(pending["SecretString"])["password"]

    # Test database connection with new credentials
    # ... connection test logic here ...


def finish_secret(arn, token):
    """Promote the new credential to current."""
    metadata = secrets_client.describe_secret(SecretId=arn)

    current_version = None
    for version_id, stages in metadata["VersionIdsToStages"].items():
        if "AWSCURRENT" in stages:
            current_version = version_id
            break

    secrets_client.update_secret_version_stage(
        SecretId=arn,
        VersionStage="AWSCURRENT",
        MoveToVersionId=token,
        RemoveFromVersionId=current_version,
    )
```

### Zero-Downtime Rotation

Zero-downtime rotation requires that applications can handle credential changes without restarting. This is achieved through dual-credential strategies or credential caching with refresh.

```
Zero-Downtime Rotation Strategy
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Time ─────────────────────────────────────────►

  Credential A:  ████████████████░░░░░░░░░░░░░░░
  Credential B:  ░░░░░░░░░░░░████████████████████

  ████ = active     ░░░░ = inactive

  Overlap period:   ░░░░████░░░░
                    Both A and B are valid
                    Application switches from A → B
                    No downtime during transition

  Key requirements:
  1. Service must accept both old and new credentials
  2. Secret rotation creates new before revoking old
  3. Applications must refresh cached credentials
  4. Health checks verify connectivity after rotation
```

```hcl
# Dual-credential pattern for zero-downtime rotation
resource "aws_secretsmanager_secret" "db_primary" {
  name = "prod/db/primary-credential"
}

resource "aws_secretsmanager_secret" "db_secondary" {
  name = "prod/db/secondary-credential"
}

# Application reads both and uses whichever is current
resource "aws_ssm_parameter" "active_credential" {
  name  = "/prod/db/active-credential-path"
  type  = "String"
  value = aws_secretsmanager_secret.db_primary.name
}
```

---

## Secrets Management Comparison

| Approach | Complexity | Security | Rotation | Audit Trail | Cost | Best For |
|---|---|---|---|---|---|---|
| terraform.tfvars | Very Low | Poor | Manual | None | Free | Local dev only |
| TF_VAR_ env vars | Low | Medium | Manual | CI/CD logs | Free | Simple CI/CD |
| SOPS encrypted files | Medium | Good | Manual | Git history | Free | Small teams, git-native workflows |
| AWS Secrets Manager | Medium | High | Automatic | CloudTrail | ~$0.40/secret/month | AWS-native teams |
| Azure Key Vault | Medium | High | Configurable | Azure Monitor | ~$0.03/10K ops | Azure-native teams |
| GCP Secret Manager | Medium | High | Configurable | Cloud Audit | ~$0.06/10K ops | GCP-native teams |
| HashiCorp Vault | High | Very High | Dynamic | Full audit | Free (OSS) / $$ (Enterprise) | Multi-cloud, advanced requirements |
| Vault Dynamic Secrets | High | Highest | Automatic | Full audit | Vault license | Zero-trust environments |

| Feature | tfvars | Env Vars | SOPS | AWS SM | Azure KV | GCP SM | Vault |
|---|---|---|---|---|---|---|---|
| Encryption at rest | ✗ | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Access control | ✗ | OS-level | Key-based | IAM | RBAC | IAM | Policy |
| Automatic rotation | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ | ✓ |
| Dynamic secrets | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| Audit logging | ✗ | ✗ | Git | ✓ | ✓ | ✓ | ✓ |
| Multi-cloud | ✓ | ✓ | ✓ | ✗ | ✗ | ✗ | ✓ |
| Terraform native | ✓ | ✓ | Plugin | Data source | Data source | Data source | Provider |
| Pulumi native | ✗ | ✓ | ✗ | SDK | SDK | SDK | Provider |
| Version history | ✗ | ✗ | Git | ✓ | ✓ | ✓ | ✓ |

> **Recommendation:** For most teams, start with environment variables for CI/CD and a cloud-native secrets manager (AWS Secrets Manager, Azure Key Vault, or GCP Secret Manager) for production secrets. Adopt SOPS for git-native encrypted files. Move to HashiCorp Vault when you need dynamic secrets, multi-cloud support, or advanced access policies.

---

## Next Steps

Continue your Infrastructure as Code learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | IaC Fundamentals | Core IaC concepts, declarative vs imperative, tool landscape |
| [01-TERRAFORM.md](01-TERRAFORM.md) | Terraform | HCL fundamentals, providers, state management, modules |
| [02-PULUMI.md](02-PULUMI.md) | Pulumi | General-purpose languages for IaC, stacks, automation API |
| [03-CLOUDFORMATION.md](03-CLOUDFORMATION.md) | AWS CloudFormation | AWS-native IaC, nested stacks, drift detection |
| [04-STATE-MANAGEMENT.md](04-STATE-MANAGEMENT.md) | State Management | Remote state, locking, import, state surgery |
| [05-MODULES-AND-REUSE.md](05-MODULES-AND-REUSE.md) | Modules and Reuse | Module design, versioning, registries |
| [06-TESTING.md](06-TESTING.md) | IaC Testing | Policy as code, plan validation, integration testing |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial Secrets and Variables documentation |
