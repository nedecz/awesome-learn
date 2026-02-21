# Infrastructure as Code Learning Path

A structured, self-paced training guide to mastering Infrastructure as Code — from foundational concepts and Terraform deep dives to multi-cloud strategies, testing, security, and production-grade patterns. Each phase builds on the previous one, progressing from core IaC principles to advanced operational practices across Terraform, Pulumi, and CloudFormation.

> **Time Estimate:** 8–10 weeks at ~5 hours/week. Adjust pace to your experience level. Engineers with prior cloud provisioning or scripting experience may complete Phase 1 in half the time.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds directly on prior knowledge; jumping ahead creates gaps that slow you down later
2. **Read the linked documents** — they contain the detailed content, code examples, and reference tables
3. **Complete the exercises** — hands-on practice is how infrastructure concepts become intuition; do not skip them
4. **Check yourself** — use the knowledge check items before moving to the next phase; if you cannot answer them confidently, re-read the relevant sections
5. **Build the capstone project** — the final project ties together all six phases and produces a portfolio artifact you can reference in engineering conversations

---

## Phase 1: Foundations (Week 1–2)

### Learning Objectives

- Understand the core concepts of Infrastructure as Code: declarative vs imperative, idempotency, and drift
- Learn the IaC tool landscape: Terraform, Pulumi, CloudFormation, and where each fits
- Grasp the infrastructure lifecycle: write, plan, apply, destroy
- Understand the benefits of IaC over manual provisioning: repeatability, version control, auditability
- Master the relationship between IaC and the broader DevOps ecosystem

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | IaC fundamentals, declarative vs imperative, tool landscape, infrastructure lifecycle, drift detection, benefits and challenges |

### Exercises

**1. IaC Concept Mapping:**

Choose a project or environment you are familiar with — a personal cloud setup, a work service, or a hypothetical three-tier web application — and map out how infrastructure is currently provisioned:

- List every piece of infrastructure (VPCs, subnets, load balancers, compute instances, databases, DNS records)
- For each resource, identify whether it was created manually (console clicks, CLI one-liners) or through automation
- Identify the feedback loops: how long does it take to know whether a change succeeded or broke something?
- Estimate the total time to recreate the entire environment from scratch
- Identify three risks of the current manual process that IaC could eliminate (drift, undocumented changes, inconsistency)

Document your findings. You will revisit this analysis after Phase 6 and compare it with your capstone infrastructure.

**2. Declarative vs Imperative Comparison:**

Examine the following two approaches to creating the same infrastructure and identify the differences:

```hcl
# Approach A: Declarative (Terraform HCL)
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

resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-${count.index + 1}"
    Tier = "public"
  }
}
```

```bash
# Approach B: Imperative (AWS CLI script)
#!/bin/bash
set -euo pipefail

VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)
aws ec2 modify-vpc-attribute --vpc-id "$VPC_ID" --enable-dns-hostnames '{"Value": true}'
aws ec2 create-tags --resources "$VPC_ID" --tags Key=Name,Value=main-vpc

SUBNET1_ID=$(aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.0.0.0/24 \
  --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
SUBNET2_ID=$(aws ec2 create-subnet --vpc-id "$VPC_ID" --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1b --query 'Subnet.SubnetId' --output text)
```

After reviewing, answer:
- What happens if you run each approach a second time without changes? Which one is idempotent?
- How does each approach handle adding a third subnet?
- Which approach makes it easier to review infrastructure changes in a pull request?
- How does each approach detect and correct drift?
- What happens if the script in Approach B fails halfway through?

**3. Tool Landscape Research:**

Research the three major IaC tools and complete the following comparison table:

```
| Feature                | Terraform          | Pulumi             | CloudFormation     |
|------------------------|--------------------|--------------------|---------------------|
| Language               |                    |                    |                     |
| State management       |                    |                    |                     |
| Cloud support          |                    |                    |                     |
| Open source            |                    |                    |                     |
| Drift detection        |                    |                    |                     |
| Module/reuse model     |                    |                    |                     |
| Learning curve         |                    |                    |                     |
| Best suited for        |                    |                    |                     |
```

For each tool, write a one-paragraph summary of when you would choose it over the others.

**4. Infrastructure Lifecycle Walkthrough:**

Set up a minimal IaC project and walk through the full lifecycle:

```bash
mkdir iac-foundations && cd iac-foundations

cat > main.tf << 'EOF'
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

resource "local_file" "hello" {
  content  = "Hello, Infrastructure as Code!"
  filename = "${path.module}/hello.txt"
}

output "file_path" {
  value = local_file.hello.filename
}
EOF

terraform init      # Initialise providers
terraform validate  # Validate configuration syntax
terraform plan      # Preview what will change
terraform apply     # Apply the changes
terraform show      # Inspect current state
terraform destroy   # Clean up all resources
```

After running through the lifecycle:
- What files did `terraform init` create? What is the `.terraform.lock.hcl` file for?
- What did `terraform plan` show you before any resources existed?
- What is the `terraform.tfstate` file, and why should it never be committed to Git?

### Knowledge Check

- [ ] What is the difference between declarative and imperative infrastructure management?
- [ ] What is idempotency, and why is it critical for Infrastructure as Code?
- [ ] What is infrastructure drift, and how do IaC tools detect and correct it?
- [ ] What are the four stages of the Terraform lifecycle (init, plan, apply, destroy)?

---

## Phase 2: Terraform Deep Dive (Week 3–4)

### Learning Objectives

- Master HashiCorp Configuration Language (HCL): resources, data sources, variables, outputs, and locals
- Understand Terraform providers and how they map to cloud APIs
- Learn state management: local state, remote backends, state locking, and import
- Implement the Terraform CLI workflow: init, plan, apply, destroy, import, taint, untaint
- Build real infrastructure with proper variable parameterisation and output references

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [01-TERRAFORM](01-TERRAFORM.md) | HCL syntax, resources, data sources, variables, outputs, locals, providers, provisioners, workspaces |
| 2 | [04-STATE-MANAGEMENT](04-STATE-MANAGEMENT.md) | State fundamentals, remote backends, state locking, state operations, import, migration, sensitive data in state |

### Exercises

**1. HCL Deep Dive — Resource Configuration:**

Examine the following Terraform configuration and identify every HCL feature used:

```hcl
variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_count" {
  description = "Number of compute instances"
  type        = number
  default     = 2
}

locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    Project     = "learning-path"
  }
  name_prefix = "myapp-${var.environment}"
}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "app" {
  count         = var.instance_count
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.environment == "prod" ? "t3.medium" : "t3.micro"

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-app-${count.index + 1}"
  })

  lifecycle {
    create_before_destroy = true
    ignore_changes        = [ami]
  }
}

output "instance_ids" {
  value = aws_instance.app[*].id
}
```

After reviewing, answer:
- What does the `validation` block on the `environment` variable do?
- What is the difference between a `variable`, a `local`, and an `output`?
- What is a `data` source, and how does it differ from a `resource`?
- What does the `lifecycle` block control, and when would you use `ignore_changes`?
- What does the splat expression `aws_instance.app[*].id` return?

**2. Remote State Configuration:**

Configure a remote backend for Terraform state management:

```hcl
# backend.tf — S3 backend with DynamoDB locking
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "environments/dev/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

```hcl
# state-infrastructure/main.tf — Create the backend resources
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state"
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default { sse_algorithm = "aws:kms" }
  }
}

resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

After reviewing:
- Why is versioning enabled on the S3 bucket? What problem does it solve?
- What is the DynamoDB table for, and what happens if two engineers run `terraform apply` simultaneously without it?
- What is the chicken-and-egg problem with creating backend resources using Terraform?
- How would you migrate from local state to this remote backend?

**3. State Operations Practice:**

Practice common state operations using the local provider (safe to run locally):

```bash
mkdir state-practice && cd state-practice

cat > main.tf << 'EOF'
terraform {
  required_providers {
    local = { source = "hashicorp/local", version = "~> 2.4" }
  }
}

resource "local_file" "config_a" {
  content  = "Configuration A"
  filename = "${path.module}/config-a.txt"
}

resource "local_file" "config_b" {
  content  = "Configuration B"
  filename = "${path.module}/config-b.txt"
}
EOF

terraform init && terraform apply -auto-approve

# Practice state operations
terraform state list                                              # List all resources
terraform state show local_file.config_a                          # Show resource details
terraform state mv local_file.config_a local_file.app_config      # Rename in state
terraform state pull > backup.tfstate                             # Export state backup
terraform plan                                                    # See what changed
```

After running:
- What happened after `terraform state mv`? Did the file on disk change?
- Why would you use `terraform state mv` when refactoring Terraform code?
- Why is backing up state before any state operation a critical safety practice?

**4. Provider Aliases for Multi-Region Setup:**

Examine the following multi-region provider configuration:

```hcl
provider "aws" {
  region = "us-east-1"
  default_tags {
    tags = { ManagedBy = "terraform", Environment = var.environment }
  }
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
  default_tags {
    tags = { ManagedBy = "terraform", Environment = var.environment }
  }
}

resource "aws_s3_bucket" "primary" {
  bucket = "myapp-data-us-east-1"
}

resource "aws_s3_bucket" "replica" {
  provider = aws.eu_west
  bucket   = "myapp-data-eu-west-1"
}
```

After reviewing:
- What does the `alias` keyword do on a provider block?
- How do you reference an aliased provider in a resource?
- What are `default_tags`, and why are they useful?
- How would you add a third region (e.g., `ap-southeast-1`)?

### Knowledge Check

- [ ] What are the six core HCL block types (terraform, provider, resource, data, variable, output)?
- [ ] What is the difference between `count` and `for_each` for creating multiple resources?
- [ ] Why must Terraform state be stored remotely with locking in team environments?
- [ ] What is `terraform import`, and when would you use it?

---

## Phase 3: Alternative Tools (Week 5–6)

### Learning Objectives

- Learn Pulumi fundamentals: projects, stacks, resources, and the use of general-purpose programming languages
- Understand AWS CloudFormation: templates, stacks, change sets, nested stacks, and drift detection
- Compare Terraform, Pulumi, and CloudFormation approaches to the same infrastructure problem
- Identify when to choose each tool based on team skills, cloud strategy, and organisational requirements

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-PULUMI](02-PULUMI.md) | Projects, stacks, resources, inputs/outputs, component resources, providers, state management, automation API |
| 2 | [03-CLOUDFORMATION](03-CLOUDFORMATION.md) | Template anatomy, resources, parameters, outputs, intrinsic functions, change sets, nested stacks, drift detection |

### Exercises

**1. Pulumi — Infrastructure with Real Code:**

Examine the following Pulumi program and compare it to the Terraform equivalent from Phase 2:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const config = new pulumi.Config();
const environment = config.require("environment");
const instanceCount = config.getNumber("instanceCount") || 2;

const vpc = new aws.ec2.Vpc("main", {
  cidrBlock: "10.0.0.0/16",
  enableDnsHostnames: true,
  enableDnsSupport: true,
  tags: { Name: `${environment}-vpc`, ManagedBy: "pulumi" },
});

const azs = aws.getAvailabilityZones({ state: "available" });
const publicSubnets: aws.ec2.Subnet[] = [];

for (let i = 0; i < 2; i++) {
  publicSubnets.push(new aws.ec2.Subnet(`public-${i}`, {
    vpcId: vpc.id,
    cidrBlock: `10.0.${i}.0/24`,
    availabilityZone: azs.then(a => a.names[i]),
    mapPublicIpOnLaunch: true,
    tags: { Name: `public-subnet-${i + 1}`, Tier: "public" },
  }));
}

export const vpcId = vpc.id;
export const subnetIds = publicSubnets.map(s => s.id);
```

After reviewing:
- How does Pulumi handle configuration values compared to Terraform variables?
- What are the advantages of using `for` loops in TypeScript versus `count` in HCL?
- What is the equivalent of `terraform plan` in Pulumi?
- How would you add unit tests for this Pulumi program using standard testing frameworks?

**2. CloudFormation — Template Anatomy:**

Examine the following CloudFormation template and identify every section:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: VPC with public subnets for the learning path project

Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, prod]

Mappings:
  InstanceTypeMap:
    dev:
      InstanceType: t3.micro
    prod:
      InstanceType: t3.medium

Conditions:
  IsProduction: !Equals [!Ref Environment, prod]

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: '10.0.0.0/16'
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-vpc'
        - Key: ManagedBy
          Value: cloudformation

  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Select [0, !Cidr ['10.0.0.0/16', 4, 8]]
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-subnet-1'

  MonitoringAlarm:
    Type: AWS::CloudWatch::Alarm
    Condition: IsProduction
    Properties:
      AlarmName: !Sub '${Environment}-high-cpu'
      MetricName: CPUUtilization
      Namespace: AWS/EC2
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 80
      ComparisonOperator: GreaterThanThreshold

Outputs:
  VpcId:
    Value: !Ref VPC
    Export:
      Name: !Sub '${AWS::StackName}-VpcId'
```

After reviewing:
- What are the six top-level sections of a CloudFormation template?
- What does the `Mappings` section provide that `Parameters` does not?
- How does the `Conditions` section work, and what resource does it gate?
- What intrinsic functions are used (`!Ref`, `!Sub`, `!Select`, `!Cidr`, `!GetAZs`, `!Equals`)?
- What does `Export` in the `Outputs` section enable?

**3. Three-Tool Comparison Exercise:**

Implement the same infrastructure in all three tools — a VPC with two public subnets and one EC2 instance. Then compare:

```
| Comparison Dimension   | Terraform          | Pulumi             | CloudFormation     |
|------------------------|--------------------|--------------------|---------------------|
| Lines of code          |                    |                    |                     |
| Time to write          |                    |                    |                     |
| Readability            |                    |                    |                     |
| Preview experience     |                    |                    |                     |
| Error messages         |                    |                    |                     |
| State management       |                    |                    |                     |
| Cleanup/destroy        |                    |                    |                     |
```

Write a one-page recommendation for which tool you would choose for:
- A startup with a small team of full-stack developers
- An enterprise with strict AWS-only policies
- A platform team managing infrastructure for 20 product teams

**4. CloudFormation Change Set Workflow:**

Practice the CloudFormation change set workflow:

```bash
# Create a stack
aws cloudformation create-stack \
  --stack-name learning-path-vpc \
  --template-body file://cloudformation-vpc.yaml \
  --parameters ParameterKey=Environment,ParameterValue=dev

# Create a change set (preview changes before applying)
aws cloudformation create-change-set \
  --stack-name learning-path-vpc \
  --change-set-name add-third-subnet \
  --template-body file://cloudformation-vpc-v2.yaml

# Review the change set
aws cloudformation describe-change-set \
  --stack-name learning-path-vpc \
  --change-set-name add-third-subnet

# Execute or delete the change set
aws cloudformation execute-change-set \
  --stack-name learning-path-vpc \
  --change-set-name add-third-subnet

# Check for drift
aws cloudformation detect-stack-drift --stack-name learning-path-vpc
```

After running:
- How does a change set compare to `terraform plan`?
- What types of changes does a change set show (Add, Modify, Remove)?
- What does drift detection reveal, and how would you resolve detected drift?

### Knowledge Check

- [ ] What is the key advantage of Pulumi over Terraform for teams with strong programming language skills?
- [ ] What are CloudFormation intrinsic functions, and why are they necessary in a YAML/JSON template?
- [ ] How does CloudFormation's change set workflow compare to `terraform plan` and `terraform apply`?
- [ ] In what scenario would you choose CloudFormation over Terraform or Pulumi?

---

## Phase 4: Reuse and Testing (Week 6–7)

### Learning Objectives

- Build reusable Terraform modules with proper input/output interfaces
- Understand module design patterns: composition, facade, and wrapper modules
- Learn module versioning and registry publication strategies
- Implement infrastructure testing: static analysis, plan validation, unit tests, and integration tests
- Build a CI/CD pipeline for infrastructure code with automated testing gates

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [05-MODULES-AND-REUSE](05-MODULES-AND-REUSE.md) | Module fundamentals, design patterns, input/output interfaces, versioning, registries, composition |
| 2 | [06-TESTING](06-TESTING.md) | Static analysis, policy as code, plan validation, unit testing, integration testing, CI/CD integration |

### Exercises

**1. Build a Reusable Networking Module:**

Create a Terraform module that provisions a VPC with configurable subnets:

```
modules/networking/
├── main.tf
├── variables.tf
├── outputs.tf
└── README.md
```

```hcl
# modules/networking/variables.tf
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Must be a valid CIDR block."
  }
}

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
}

variable "public_subnet_count" {
  description = "Number of public subnets"
  type        = number
  default     = 2
}

variable "private_subnet_count" {
  description = "Number of private subnets"
  type        = number
  default     = 2
}

variable "tags" {
  description = "Additional tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/networking/main.tf
data "aws_availability_zones" "available" { state = "available" }

locals {
  az_count = min(var.public_subnet_count, length(data.aws_availability_zones.available.names))
  common_tags = merge(var.tags, {
    Module      = "networking"
    Environment = var.environment
    ManagedBy   = "terraform"
  })
}

resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = merge(local.common_tags, { Name = "${var.environment}-vpc" })
}

resource "aws_subnet" "public" {
  count             = var.public_subnet_count
  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index % local.az_count]
  map_public_ip_on_launch = true
  tags = merge(local.common_tags, { Name = "${var.environment}-public-${count.index + 1}", Tier = "public" })
}

resource "aws_subnet" "private" {
  count             = var.private_subnet_count
  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index % local.az_count]
  tags = merge(local.common_tags, { Name = "${var.environment}-private-${count.index + 1}", Tier = "private" })
}
```

After building the module:
- Call the module from a root configuration with different parameter combinations
- Test that input validation rejects invalid values
- Document the module interface: what inputs are required, what outputs are available
- Explain when you would publish this to a private Terraform registry

**2. Module Composition — Building a Full Stack:**

Compose multiple modules to build a complete application stack:

```hcl
# environments/dev/main.tf
module "networking" {
  source              = "../../modules/networking"
  vpc_cidr            = "10.0.0.0/16"
  environment         = "dev"
  public_subnet_count = 2
  private_subnet_count = 2
  tags = { Project = "learning-path", Owner = "platform-team" }
}

module "compute" {
  source         = "../../modules/compute"
  environment    = "dev"
  vpc_id         = module.networking.vpc_id
  subnet_ids     = module.networking.private_subnet_ids
  instance_count = 1
  instance_type  = "t3.micro"
}

module "database" {
  source            = "../../modules/database"
  environment       = "dev"
  vpc_id            = module.networking.vpc_id
  subnet_ids        = module.networking.private_subnet_ids
  instance_class    = "db.t3.micro"
  allocated_storage = 20
}
```

Design the `compute` and `database` modules yourself:
- Define the required variables, resources, and outputs
- Ensure modules communicate only through outputs (no hardcoded cross-references)
- Add input validation for critical parameters
- Write a `README.md` for each module with usage examples

**3. Infrastructure Testing Pipeline:**

Design a testing pipeline that validates infrastructure code at multiple levels:

```yaml
# .github/workflows/infra-test.yml
name: Infrastructure Tests
on:
  pull_request:
    paths: ['modules/**', 'environments/**', '*.tf']

jobs:
  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Terraform Format Check
        run: terraform fmt -check -recursive
      - name: TFLint
        uses: terraform-linters/setup-tflint@v4
      - run: tflint --init && tflint --recursive
      - name: Checkov Security Scan
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: .
          framework: terraform

  plan-validation:
    runs-on: ubuntu-latest
    needs: static-analysis
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: |
          cd environments/dev
          terraform init -backend=false
          terraform validate
          terraform plan -out=plan.tfplan
      - name: Policy Check (OPA)
        run: |
          terraform show -json plan.tfplan > plan.json
          opa eval --data policies/ --input plan.json "data.terraform.deny[msg]"
        working-directory: environments/dev

  integration-tests:
    runs-on: ubuntu-latest
    needs: plan-validation
    steps:
      - uses: actions/checkout@v4
      - name: Run Terratest
        run: cd tests/ && go test -v -timeout 30m ./...
```

For this pipeline:
- Explain what each stage validates and what types of issues it catches
- What is the difference between `terraform fmt -check` and `terraform validate`?
- How does Checkov differ from OPA for policy enforcement?
- What is Terratest, and why does it need a 30-minute timeout?

**4. Write a Terratest for the Networking Module:**

Examine the following Go test and understand how Terratest validates infrastructure:

```go
package tests

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestNetworkingModule(t *testing.T) {
    t.Parallel()

    opts := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../modules/networking",
        Vars: map[string]interface{}{
            "environment":          "dev",
            "vpc_cidr":             "10.99.0.0/16",
            "public_subnet_count":  2,
            "private_subnet_count": 2,
            "tags":                 map[string]string{"Test": "true"},
        },
    })

    defer terraform.Destroy(t, opts)
    terraform.InitAndApply(t, opts)

    vpcId := terraform.Output(t, opts, "vpc_id")
    assert.NotEmpty(t, vpcId, "VPC ID should not be empty")

    publicSubnetIds := terraform.OutputList(t, opts, "public_subnet_ids")
    assert.Equal(t, 2, len(publicSubnetIds), "Should have 2 public subnets")

    privateSubnetIds := terraform.OutputList(t, opts, "private_subnet_ids")
    assert.Equal(t, 2, len(privateSubnetIds), "Should have 2 private subnets")
}
```

After reviewing:
- What does `defer terraform.Destroy` ensure, and why is it important?
- Why does the test use a unique CIDR (`10.99.0.0/16`) instead of the default?
- What does `t.Parallel()` do, and what are the implications for state isolation?
- What cloud resources are created during this test, and what is the cost implication?

### Knowledge Check

- [ ] What makes a good module interface (clear inputs, validated types, documented outputs)?
- [ ] What is the difference between static analysis (fmt, validate, lint) and integration testing (Terratest)?
- [ ] What is policy as code, and how does it prevent non-compliant infrastructure from being deployed?
- [ ] Why should modules communicate only through outputs, never through hardcoded resource references?

---

## Phase 5: Security and Scale (Week 7–8)

### Learning Objectives

- Implement secrets management in IaC: environment variables, encrypted files, vault integration, and dynamic secrets
- Understand variable hierarchies and environment-specific configuration patterns
- Learn multi-cloud IaC strategies: abstraction layers, provider-specific modules, and unified tooling
- Design infrastructure for multiple cloud providers with consistent patterns

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [07-SECRETS-AND-VARIABLES](07-SECRETS-AND-VARIABLES.md) | Secrets problem, variable hierarchies, environment configuration, vault integration, SOPS, dynamic secrets, state security |
| 2 | [08-MULTI-CLOUD](08-MULTI-CLOUD.md) | Business drivers, architecture patterns, abstraction layers, provider modules, operational practices |

### Exercises

**1. Secrets Management Audit:**

Examine the following Terraform configuration and identify every security issue:

```hcl
# BAD EXAMPLE — find all the security problems
variable "db_password" {
  default = "SuperSecret123!"
}

resource "aws_db_instance" "main" {
  engine              = "postgres"
  instance_class      = "db.t3.medium"
  username            = "admin"
  password            = var.db_password
  publicly_accessible = true
  skip_final_snapshot = true
  storage_encrypted   = false
}

resource "aws_ssm_parameter" "db_connection" {
  name  = "/myapp/db-url"
  type  = "String"
  value = "postgres://admin:${var.db_password}@${aws_db_instance.main.endpoint}/myapp"
}

output "db_password" {
  value = var.db_password
}
```

Write the corrected version addressing every issue:

```hcl
# CORRECTED — fix every issue
variable "db_password" {
  description = "Database master password"
  type        = string
  sensitive   = true
  # No default — must be provided at runtime

  validation {
    condition     = length(var.db_password) >= 16
    error_message = "Password must be at least 16 characters."
  }
}

resource "aws_db_instance" "main" {
  engine              = "postgres"
  instance_class      = "db.t3.medium"
  username            = "admin"
  password            = var.db_password
  publicly_accessible = false          # Fixed: no public access
  skip_final_snapshot = false          # Fixed: keep final snapshot
  storage_encrypted   = true           # Fixed: encrypt at rest
  deletion_protection = true           # Added: prevent accidental deletion
}

resource "aws_ssm_parameter" "db_connection" {
  name  = "/myapp/db-url"
  type  = "SecureString"               # Fixed: encrypt the parameter
  value = "postgres://admin:${var.db_password}@${aws_db_instance.main.endpoint}/myapp"
}

# Removed: db_password output — never expose secrets in outputs
```

For each fix, explain:
- Why the original code was insecure
- The potential impact if the vulnerability were exploited
- Whether the issue would appear in state, logs, or version control

**2. Vault Integration for Dynamic Secrets:**

Examine the following Vault-integrated Terraform configuration:

```hcl
provider "vault" {
  address = "https://vault.example.com:8200"
}

data "vault_generic_secret" "db_creds" {
  path = "database/creds/myapp-role"
}

resource "aws_db_instance" "main" {
  engine            = "postgres"
  instance_class    = "db.t3.medium"
  username          = data.vault_generic_secret.db_creds.data["username"]
  password          = data.vault_generic_secret.db_creds.data["password"]
  publicly_accessible = false
  storage_encrypted   = true
}
```

After reviewing:
- What are dynamic secrets, and how do they differ from static secrets?
- Why is reading secrets from Vault at apply time better than storing them in Terraform variables?
- What happens to the Vault credential when `terraform destroy` is run?

**3. Multi-Cloud Architecture Design:**

Design a multi-cloud infrastructure setup for a disaster recovery scenario:

```
Requirements:
- Primary: AWS (us-east-1) — all production workloads
- Secondary: Azure (East US) — disaster recovery and failover
- DNS failover must route traffic to the secondary site within 5 minutes
- Infrastructure managed from a single Terraform codebase
```

Create the directory structure:

```
multi-cloud/
├── modules/
│   ├── aws-networking/
│   ├── azure-networking/
│   ├── aws-compute/
│   └── azure-compute/
├── environments/
│   ├── primary/          # AWS
│   └── secondary/        # Azure
└── global/
    └── dns-failover/
```

For your design:
- How do you maintain consistency between AWS and Azure when the APIs differ?
- Where does the DNS failover configuration live?
- How do you handle state management — one state file or separate state per environment?
- What are the trade-offs between provider-specific modules and a cloud-agnostic abstraction layer?

**4. Environment-Specific Configuration:**

Implement a variable hierarchy for three environments using `tfvars` files:

```hcl
# environments/dev.tfvars
environment                = "dev"
instance_type              = "t3.micro"
instance_count             = 1
enable_monitoring          = false
enable_deletion_protection = false
db_instance_class          = "db.t3.micro"
db_multi_az                = false
```

```hcl
# environments/prod.tfvars
environment                = "prod"
instance_type              = "t3.medium"
instance_count             = 3
enable_monitoring          = true
enable_deletion_protection = true
db_instance_class          = "db.r6g.large"
db_multi_az                = true
```

After configuring:
- How do you select which `tfvars` file to use at apply time?
- What values differ between environments, and what pattern determines the differences?
- How would you add a CI/CD pipeline that applies the correct `tfvars` per environment?
- How does this pattern scale to 10+ environments or 50+ variables?

### Knowledge Check

- [ ] What are three places secrets can accidentally leak in IaC (state files, logs, version control)?
- [ ] What is the difference between static secrets and dynamic secrets (generated on demand)?
- [ ] What is the primary challenge of multi-cloud IaC, and how do abstraction layers help?
- [ ] How do `tfvars` files enable environment-specific configuration without code duplication?

---

## Phase 6: Production Readiness (Week 9–10)

### Learning Objectives

- Apply IaC best practices across directory structure, naming conventions, CI/CD integration, and documentation
- Implement drift management, blast radius reduction, and operational readiness checklists
- Identify and remediate common IaC anti-patterns
- Build production-grade infrastructure with proper tagging, environment management, and change control

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Directory structure, naming conventions, code organisation, version pinning, CI/CD integration, state management, tagging, documentation, drift management |
| 2 | [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | Common IaC anti-patterns, quick reference checklist |

### Exercises

**1. Production Readiness Checklist Audit:**

Evaluate a Terraform codebase against the following production readiness checklist:

```
Infrastructure Code Quality:
  [ ] All resources have consistent naming conventions
  [ ] All provider and module versions are pinned
  [ ] terraform fmt and terraform validate pass
  [ ] No hardcoded values — all environment-specific values in variables
  [ ] All variables have descriptions, types, and validation where appropriate

State Management:
  [ ] State is stored remotely with locking enabled
  [ ] State bucket has versioning and encryption enabled
  [ ] No secrets stored in variable defaults or committed tfvars

Security:
  [ ] No resources publicly accessible unless explicitly required
  [ ] All databases have encryption at rest and deletion protection (prod)
  [ ] Security groups follow least-privilege
  [ ] Secrets sourced from a secrets manager, not from variables

Tagging:
  [ ] All resources have Environment, ManagedBy, Project, and Owner tags
  [ ] Tags applied via locals or default_tags, not repeated per resource

Documentation:
  [ ] README at root with architecture overview
  [ ] Each module has a README with inputs, outputs, and usage examples
  [ ] Runbook for common operations (apply, rollback, import, state recovery)
```

For each unchecked item, write a one-paragraph remediation plan: what to change, estimated effort, and priority.

**2. Anti-Pattern Identification Exercise:**

Review the following system description and identify at least 6 anti-patterns from [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md):

> "Our team manages infrastructure for 8 microservices. Each service has its own Terraform configuration in a subdirectory, but they all share a single state file stored locally on the lead engineer's laptop. Provider versions are not pinned — we always use 'latest' so we get the newest features. There are no modules; each service copies and pastes the VPC and subnet configuration, so we have 8 slightly different VPC definitions. We never run `terraform plan` before `terraform apply` because it takes too long. Secrets like database passwords are stored as default values in `variables.tf` and committed to Git. Our naming convention is inconsistent — some resources use hyphens, others use underscores, and some have no name tags at all. When a resource needs to be changed, we modify it in the AWS console first, then update the Terraform code later. There are no tests, no linting, and no CI/CD pipeline for infrastructure changes. The production and development environments use completely different Terraform configurations written by different engineers."

For each anti-pattern:
- State the anti-pattern name from the document
- Quote the specific evidence from the description
- Write the recommended fix in 2–3 sentences

<details>
<summary>Anti-patterns identified (click to reveal suggested answers)</summary>

1. **Local/Shared State** — "single state file stored locally on the lead engineer's laptop"
2. **Unpinned Providers** — "Provider versions are not pinned — we always use 'latest'"
3. **Copy-Paste Infrastructure** — "each service copies and pastes the VPC and subnet configuration"
4. **Skipping Plan** — "We never run terraform plan before terraform apply"
5. **Secrets in Code** — "database passwords are stored as default values in variables.tf and committed to Git"
6. **Inconsistent Naming** — "some resources use hyphens, others use underscores, and some have no name tags"
7. **ClickOps Before Code** — "we modify it in the AWS console first, then update the Terraform code later"
8. **No Testing or CI/CD** — "no tests, no linting, and no CI/CD pipeline for infrastructure changes"
9. **Environment Divergence** — "production and development environments use completely different Terraform configurations"

</details>

**3. Drift Detection and Remediation:**

Practice detecting and resolving infrastructure drift:

```bash
mkdir drift-practice && cd drift-practice

cat > main.tf << 'EOF'
terraform {
  required_providers {
    local = { source = "hashicorp/local", version = "~> 2.4" }
  }
}

resource "local_file" "config" {
  content  = "original content"
  filename = "${path.module}/config.txt"
}
EOF

terraform init && terraform apply -auto-approve

# Simulate drift — manually modify the file outside Terraform
echo "manually modified content" > config.txt

# Detect drift
terraform plan
# Terraform will show that the file content has changed

# Remediation options:
# Option A: Overwrite the manual change (re-apply Terraform)
terraform apply -auto-approve

# Option B: Accept the manual change (update Terraform to match)
# Edit main.tf to set content = "manually modified content"
```

After running:
- What did `terraform plan` show after the manual modification?
- When would you choose Option A (overwrite) versus Option B (accept)?
- What process should a team follow when drift is detected in production?
- How would you set up automated drift detection on a schedule?

**4. Tagging Strategy Design:**

Design a comprehensive tagging strategy for a multi-team organisation:

```hcl
locals {
  required_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    Project     = var.project_name
    Owner       = var.team_name
    CostCenter  = var.cost_center
  }

  optional_tags = {
    Application = var.application_name
    Compliance  = var.compliance_framework
    DataClass   = var.data_classification
    BackupPolicy = var.backup_policy
  }
}

provider "aws" {
  region = var.region
  default_tags {
    tags = local.required_tags
  }
}
```

Design your own tagging strategy:
- Define which tags are required versus optional
- Explain how tags enable cost allocation and chargeback
- Show how to enforce tagging with a policy-as-code tool (OPA, Sentinel, or Checkov)
- Document your tagging standard in a one-page reference for other engineers

### Knowledge Check

- [ ] What are the five areas of a production readiness checklist (code quality, state, security, tagging, documentation)?
- [ ] What is infrastructure drift, and what are three strategies for handling it?
- [ ] Name three anti-patterns from the checklist and explain why each is harmful
- [ ] Why is a consistent tagging strategy important for cost management and compliance?

---

## Capstone Project

### Project Overview

Design and implement a multi-environment (dev/staging/prod) infrastructure setup using Terraform that demonstrates production-grade practices from all six phases. Build a complete infrastructure platform for a web application with networking, compute, database, and supporting services.

### System Architecture

```
                ┌─────────────────────────────────────────┐
                │         Git Repository                   │
                │  (infrastructure code + CI/CD pipeline)  │
                └──────────────────┬──────────────────────┘
                                   │
                        push / pull_request
                                   │
                ┌──────────────────▼──────────────────────┐
                │         CI Pipeline                      │
                │                                          │
                │  ┌─────────────┐  ┌──────────────────┐  │
                │  │ Format/Lint │→ │ Validate/TFSec   │  │
                │  └─────────────┘  └───────┬──────────┘  │
                │                           │              │
                │  ┌────────────────────────▼──────────┐  │
                │  │ Terraform Plan + Policy Check      │  │
                │  └───────┬───────────────────────────┘  │
                │          │                               │
                │  ┌───────▼──────┐  ┌─────────────────┐  │
                │  │ Cost         │  │ Security         │  │
                │  │ Estimation   │  │ Scanning         │  │
                │  └───────┬──────┘  └┬────────────────┘  │
                │          │          │                    │
                │  ┌───────▼──────────▼────────────────┐  │
                │  │ Plan Artifact Upload               │  │
                │  └──────────────┬────────────────────┘  │
                └─────────────────┼────────────────────────┘
                                  │
                ┌─────────────────▼────────────────────────┐
                │         CD Pipeline                       │
                │                                           │
                │  ┌────────────────────────────────────┐  │
                │  │ Apply to Dev (automatic)            │  │
                │  └──────────────┬─────────────────────┘  │
                │                 │                         │
                │  ┌──────────────▼─────────────────────┐  │
                │  │ Apply to Staging (automatic)        │  │
                │  └──────────────┬─────────────────────┘  │
                │                 │                         │
                │  ┌──────────────▼─────────────────────┐  │
                │  │ Manual Approval Gate                │  │
                │  └──────────────┬─────────────────────┘  │
                │                 │                         │
                │  ┌──────────────▼─────────────────────┐  │
                │  │ Apply to Production                 │  │
                │  └──────────────┬─────────────────────┘  │
                │                 │                         │
                │  ┌──────────────▼─────────────────────┐  │
                │  │ Post-Apply Validation               │  │
                │  └────────────────────────────────────┘  │
                └──────────────────────────────────────────┘
```

### Requirements

| Requirement | What to Implement |
|-------------|-------------------|
| **Remote State** | S3 backend with DynamoDB locking; separate state files per environment; versioning and encryption |
| **Reusable Modules** | Networking module (VPC, subnets, NAT, IGW), compute module (ASG, ALB, launch template), database module (RDS, subnet group, security group) |
| **Secret Management** | No secrets in code or state defaults; integrate AWS Secrets Manager or HashiCorp Vault; mark sensitive variables |
| **CI/CD Pipeline** | Lint, validate, plan on PR; apply on merge to main; separate workflows per environment |
| **Policy-as-Code** | OPA or Checkov policies enforcing tagging, encryption, no public access, approved instance types |
| **Tagging Strategy** | All resources tagged with Environment, ManagedBy, Project, Owner, CostCenter; enforced via default_tags and policy |
| **Environment Parity** | Same Terraform code for all environments; differences expressed only through tfvars files |
| **Documentation** | Root README with architecture diagram; module READMEs; runbook for operations |
| **Anti-Pattern Audit** | Run the quick reference checklist from [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md); zero anti-patterns in final submission |

### Implementation Steps

**Step 1: Set Up the Repository Structure**

```
infra-capstone/
├── .github/workflows/
│   ├── ci.yml
│   └── cd.yml
├── modules/
│   ├── networking/
│   │   ├── main.tf, variables.tf, outputs.tf, README.md
│   ├── compute/
│   │   ├── main.tf, variables.tf, outputs.tf, README.md
│   └── database/
│       ├── main.tf, variables.tf, outputs.tf, README.md
├── environments/
│   ├── dev/
│   │   ├── main.tf, backend.tf, variables.tf, terraform.tfvars
│   ├── staging/
│   │   ├── main.tf, backend.tf, variables.tf, terraform.tfvars
│   └── prod/
│       ├── main.tf, backend.tf, variables.tf, terraform.tfvars
├── policies/
│   ├── tagging.rego
│   └── security.rego
├── tests/
│   ├── networking_test.go
│   └── compute_test.go
├── docs/
│   ├── architecture.md
│   └── runbook.md
└── README.md
```

**Step 2: Build the Modules**

Each module must have a clear input interface with validation rules, produce outputs that other modules can consume, use `locals` for common tags and naming, and include no hardcoded values.

- **Networking:** VPC, public/private subnets, Internet Gateway, optional NAT Gateway, route tables
- **Compute:** Auto Scaling Group, Application Load Balancer, launch template, security groups
- **Database:** RDS instance, DB subnet group, security group, encryption, backups, deletion protection (prod only)

**Step 3: Configure Environments**

Each environment uses the same modules with different `tfvars` values, its own remote state backend key, and appropriate scaling (1 instance in dev, 2 in staging, 3+ in prod).

**Step 4: Implement CI/CD**

The CI pipeline (on pull request) must run format check, lint, validate, plan, security scan, and policy check. The CD pipeline (on merge to main) must apply to dev automatically, staging automatically after dev, and require manual approval before production.

**Step 5: Add Policy-as-Code**

Write at least two OPA or Checkov policies:

```rego
# policies/tagging.rego
package terraform.tagging

required_tags := {"Environment", "ManagedBy", "Project", "Owner", "CostCenter"}

deny[msg] {
    resource := input.planned_values.root_module.resources[_]
    tags := object.get(resource.values, "tags", {})
    missing := required_tags - {key | tags[key]}
    count(missing) > 0
    msg := sprintf("Resource %s missing tags: %v", [resource.address, missing])
}
```

```rego
# policies/security.rego
package terraform.security

deny[msg] {
    resource := input.planned_values.root_module.resources[_]
    resource.type == "aws_db_instance"
    resource.values.publicly_accessible == true
    msg := sprintf("Database %s must not be publicly accessible", [resource.address])
}

deny[msg] {
    resource := input.planned_values.root_module.resources[_]
    resource.type == "aws_db_instance"
    resource.values.storage_encrypted != true
    msg := sprintf("Database %s must have encryption at rest", [resource.address])
}
```

**Step 6: Document Everything**

- **README.md** — Architecture overview, prerequisites, quick start
- **docs/architecture.md** — Module relationships, data flow diagram
- **docs/runbook.md** — How to apply changes, roll back failures, recover state, import resources, rotate secrets

### Evaluation Criteria

| Area | What to Verify |
|------|----------------|
| **Module Design** | Modules have clear interfaces, validation rules, and documentation; no hardcoded values |
| **State Management** | Remote state with locking; separate state per environment; versioning and encryption |
| **Security** | No secrets in code; encryption at rest; least-privilege networking; policy-as-code enforced |
| **CI/CD** | Automated lint, validate, plan on PR; automated apply with approval gates for production |
| **Environment Parity** | Same code deployed to all environments; differences expressed only through tfvars |
| **Tagging** | Consistent tagging via default_tags; cost allocation and compliance tags present |
| **Anti-Patterns** | All anti-patterns from checklist evaluated; zero anti-patterns in final submission |
| **Documentation** | README, architecture diagram, runbook, and module documentation all present and accurate |

---

## Reference Table

| # | Document | Phase | Key Topics |
|---|----------|-------|------------|
| 00 | [OVERVIEW](00-OVERVIEW.md) | 1 | IaC fundamentals, declarative vs imperative, tool landscape, infrastructure lifecycle |
| 01 | [TERRAFORM](01-TERRAFORM.md) | 2 | HCL syntax, resources, data sources, variables, providers, workspaces |
| 02 | [PULUMI](02-PULUMI.md) | 3 | Projects, stacks, component resources, automation API, general-purpose languages |
| 03 | [CLOUDFORMATION](03-CLOUDFORMATION.md) | 3 | Templates, stacks, change sets, nested stacks, intrinsic functions, drift detection |
| 04 | [STATE-MANAGEMENT](04-STATE-MANAGEMENT.md) | 2 | Remote backends, state locking, state operations, import, sensitive data in state |
| 05 | [MODULES-AND-REUSE](05-MODULES-AND-REUSE.md) | 4 | Module design, composition, versioning, registries, cross-team sharing |
| 06 | [TESTING](06-TESTING.md) | 4 | Static analysis, policy as code, plan validation, unit/integration/E2E testing |
| 07 | [SECRETS-AND-VARIABLES](07-SECRETS-AND-VARIABLES.md) | 5 | Variable hierarchies, vault integration, SOPS, dynamic secrets, state security |
| 08 | [MULTI-CLOUD](08-MULTI-CLOUD.md) | 5 | Business drivers, architecture patterns, abstraction layers, operational practices |
| 09 | [BEST-PRACTICES](09-BEST-PRACTICES.md) | 6 | Directory structure, naming, CI/CD integration, tagging, drift management, documentation |
| 10 | [ANTI-PATTERNS](10-ANTI-PATTERNS.md) | 6 | Common IaC anti-patterns, quick reference checklist |
| — | [LEARNING-PATH](LEARNING-PATH.md) | All | This document — structured 6-phase curriculum |

---

## Next Steps

1. **Complete the capstone** — The capstone project is the most important part of this learning path; it ties together every phase into a single deliverable.
2. **Audit your existing infrastructure** — Apply the [anti-patterns checklist](10-ANTI-PATTERNS.md) and [best practices checklist](09-BEST-PRACTICES.md) to your production infrastructure code.
3. **Learn CI/CD** — Infrastructure code needs automated pipelines for plan, test, and apply. See the [CI/CD learning materials](../ci-cd/) in this repository.
4. **Explore Kubernetes** — Many IaC pipelines deploy to Kubernetes clusters. See the [Kubernetes learning materials](../kubernetes/) in this repository.
5. **Contribute** — Found an error, have a better example, or want to add a new exercise? Open a PR following the [CONTRIBUTING.md](../CONTRIBUTING.md) guide.

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026 | Initial IaC Learning Path documentation |
