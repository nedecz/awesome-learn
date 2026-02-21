# Testing Infrastructure as Code

A comprehensive guide to testing infrastructure as code — covering static analysis, policy as code, plan validation, unit testing, integration testing, end-to-end testing, contract testing, cost testing, security scanning, and CI/CD pipeline integration for reliable, safe, and repeatable infrastructure changes.

---

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Why Test Infrastructure Code](#why-test-infrastructure-code)
   - [Risk of Untested Infrastructure](#risk-of-untested-infrastructure)
   - [Cost of Mistakes](#cost-of-mistakes)
   - [Shift-Left for Infrastructure](#shift-left-for-infrastructure)
3. [The IaC Testing Pyramid](#the-iac-testing-pyramid)
   - [Layers of the Pyramid](#layers-of-the-pyramid)
   - [Balancing Speed and Confidence](#balancing-speed-and-confidence)
4. [Static Analysis](#static-analysis)
   - [Terraform Validate and Format](#terraform-validate-and-format)
   - [TFLint](#tflint)
   - [Checkov](#checkov)
   - [tfsec and Trivy](#tfsec-and-trivy)
   - [cfn-lint](#cfn-lint)
5. [Policy as Code](#policy-as-code)
   - [Open Policy Agent (OPA) with Rego](#open-policy-agent-opa-with-rego)
   - [HashiCorp Sentinel](#hashicorp-sentinel)
   - [AWS Config Rules](#aws-config-rules)
   - [Azure Policy](#azure-policy)
6. [Plan Validation](#plan-validation)
   - [Terraform Plan Analysis](#terraform-plan-analysis)
   - [Plan Assertions](#plan-assertions)
   - [Plan-Based Testing](#plan-based-testing)
   - [Cost Estimation Validation](#cost-estimation-validation)
7. [Unit Testing](#unit-testing)
   - [Terraform Test Framework](#terraform-test-framework)
   - [Pulumi Unit Tests](#pulumi-unit-tests)
   - [CDK Assertions](#cdk-assertions)
8. [Integration Testing](#integration-testing)
   - [Terratest](#terratest)
   - [kitchen-terraform](#kitchen-terraform)
   - [Test Fixtures and Cleanup](#test-fixtures-and-cleanup)
9. [End-to-End Testing](#end-to-end-testing)
   - [Full Stack Deployment Testing](#full-stack-deployment-testing)
   - [Smoke Tests](#smoke-tests)
   - [Cross-Service Validation](#cross-service-validation)
10. [Contract Testing](#contract-testing)
    - [Module Interface Contracts](#module-interface-contracts)
    - [Provider Contract Testing](#provider-contract-testing)
    - [Cross-Team Module Validation](#cross-team-module-validation)
11. [Cost Testing](#cost-testing)
    - [Infracost](#infracost)
    - [Cost Policies](#cost-policies)
    - [Budget Alerts as Code](#budget-alerts-as-code)
12. [Security Scanning](#security-scanning)
    - [Checkov Deep Dive](#checkov-deep-dive)
    - [Snyk IaC](#snyk-iac)
    - [KICS](#kics)
    - [Bridgecrew](#bridgecrew)
13. [Testing in CI/CD Pipelines](#testing-in-cicd-pipelines)
    - [GitHub Actions Pipeline](#github-actions-pipeline)
    - [Azure DevOps Pipeline](#azure-devops-pipeline)
    - [Quality Gates](#quality-gates)
14. [Test Strategy Comparison](#test-strategy-comparison)
15. [Next Steps](#next-steps)
16. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **testing infrastructure as code** — the practices, tools, and strategies that enable teams to validate infrastructure changes before they reach production. Infrastructure testing spans a wide spectrum, from fast static analysis that catches syntax errors and misconfigurations in seconds, to full end-to-end tests that deploy real infrastructure and verify its behavior. A well-designed IaC testing strategy catches defects early, enforces organizational policies, controls costs, and builds confidence that infrastructure changes are safe to apply.

### Target Audience

- **DevOps Engineers** building and maintaining Terraform, Pulumi, or CloudFormation configurations who need confidence that changes are safe
- **Platform Engineers** designing reusable modules and golden paths that must be validated before publishing to internal registries
- **Site Reliability Engineers (SREs)** responsible for infrastructure reliability and who need automated guardrails against misconfigurations
- **Security Engineers** enforcing compliance and security policies across infrastructure definitions using policy-as-code frameworks
- **Developers** contributing to infrastructure code alongside application code and participating in infrastructure pull request reviews

### Scope

- Why testing infrastructure code is essential and what happens when it is skipped
- The IaC testing pyramid from static analysis through end-to-end tests
- Static analysis tools: terraform validate, tflint, checkov, tfsec, trivy, cfn-lint
- Policy as code with OPA/Rego, Sentinel, AWS Config, and Azure Policy
- Plan validation and plan-based assertions
- Unit testing with the Terraform test framework, Pulumi mocks, and CDK assertions
- Integration testing with Terratest and kitchen-terraform
- End-to-end, contract, cost, and security testing
- CI/CD pipeline integration with GitHub Actions and Azure DevOps

---

## Why Test Infrastructure Code

### Risk of Untested Infrastructure

Infrastructure code that is deployed without testing carries risks that are fundamentally different from untested application code. A bug in application code might cause a user-facing error. A bug in infrastructure code can delete a production database, expose private networks to the internet, create resources that cost thousands of dollars per hour, or violate regulatory compliance requirements.

```
Untested Infrastructure Risks
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Misconfigured security group          Exposed database to internet
          │                                      │
          ▼                                      ▼
  ┌──────────────┐                      ┌──────────────┐
  │  Data Breach  │                      │  Compliance  │
  │               │                      │  Violation   │
  └──────────────┘                      └──────────────┘

  Unvalidated resource sizing           Orphaned resources
          │                                      │
          ▼                                      ▼
  ┌──────────────┐                      ┌──────────────┐
  │  Cost        │                      │  Budget      │
  │  Explosion   │                      │  Overrun     │
  └──────────────┘                      └──────────────┘
```

Common failures from untested infrastructure:

| Failure Category | Example | Impact |
|---|---|---|
| Security | S3 bucket with public access | Data breach, regulatory fines |
| Networking | Missing firewall rule | Service unreachable or overexposed |
| Sizing | Oversized instance types | Thousands in wasted spend per month |
| Dependencies | Wrong resource ordering | Failed deployments, partial state |
| Naming | Duplicate resource names | Terraform apply errors, conflicts |
| State | Missing state lock configuration | Concurrent modifications, corruption |

### Cost of Mistakes

Infrastructure mistakes are expensive in multiple dimensions. Direct costs include cloud spend for misconfigured resources, incident response time, and potential regulatory penalties. Indirect costs include lost customer trust, engineering time spent on rollbacks, and organizational fear of making infrastructure changes.

The cost to fix an infrastructure defect increases dramatically the later it is caught:

| Stage | Detection Cost | Fix Cost | Risk Level |
|---|---|---|---|
| Code review | Low | Low | Minimal |
| Static analysis | Very low | Very low | None |
| Plan review | Low | Low | None |
| Apply to staging | Medium | Medium | Low |
| Apply to production | High | High | High |
| Post-incident | Very high | Very high | Critical |

### Shift-Left for Infrastructure

Shift-left testing means moving validation as early as possible in the development lifecycle. For infrastructure code, this means catching misconfigurations at the time code is written — not when it is applied to a live environment.

Shift-left practices for infrastructure:

- **Editor integration** — Run linters and formatters on save to catch syntax and style issues immediately
- **Pre-commit hooks** — Run terraform validate, terraform fmt, and tflint before code is committed
- **Pull request checks** — Run static analysis, policy checks, and plan validation on every pull request
- **Automated plan comments** — Post terraform plan output as a PR comment for human review
- **Cost estimation** — Show estimated cost impact in pull requests before merge

```
Shift-Left Timeline
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Editor        Pre-commit     PR Checks      Staging         Production
    │               │              │              │                │
    ▼               ▼              ▼              ▼                ▼
  ┌─────┐      ┌─────────┐   ┌──────────┐   ┌─────────┐    ┌──────────┐
  │Lint │      │Validate │   │Plan +    │   │Deploy + │    │  Apply   │
  │+Fmt │      │+Format  │   │Policy +  │   │Integ.   │    │  Final   │
  │     │      │+Lint    │   │Cost Est. │   │Tests    │    │          │
  └─────┘      └─────────┘   └──────────┘   └─────────┘    └──────────┘
  Seconds       Seconds       Minutes        Minutes         Minutes

  ◀━━━━━━━━━━━━━━━  Cheaper to fix  ━━━━━━━━━━━━━━━━━━━━━━▶
  ◀━━━━━━━━━━━━━━━  Faster feedback  ━━━━━━━━━━━━━━━━━━━━━▶
```

---

## The IaC Testing Pyramid

### Layers of the Pyramid

The IaC testing pyramid mirrors the traditional software testing pyramid but is adapted for infrastructure concerns. The base consists of fast, cheap, static checks. Each layer above adds more confidence but at the cost of speed and complexity.

```
IaC Testing Pyramid
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

                        ╱╲
                       ╱  ╲              End-to-End Tests
                      ╱ E2E╲             Full stack deploy + verify
                     ╱──────╲            Minutes | High confidence
                    ╱        ╲
                   ╱Integration╲         Integration Tests
                  ╱   Tests     ╲        Real infra deploy + test
                 ╱───────────────╲       Minutes | Real resources
                ╱                 ╲
               ╱   Unit Tests      ╲     Unit Tests
              ╱                     ╲    Mock providers, test logic
             ╱───────────────────────╲   Seconds | No real resources
            ╱                         ╲
           ╱    Plan Validation        ╲  Plan Validation
          ╱                             ╲ Analyze plan output
         ╱───────────────────────────────╲Seconds | No apply
        ╱                                 ╲
       ╱      Static Analysis + Policy     ╲ Static Analysis
      ╱                                     ╲ Lint, validate, policy
     ╱───────────────────────────────────────╲ Seconds | No init
    ╱                                         ╲
   ╱━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╲

   ◀━━━━ More tests, faster, cheaper ━━━━━━━━━━━▶
   ◀━━━━ Fewer tests, slower, more confidence ━━▶
```

### Balancing Speed and Confidence

Each layer in the pyramid serves a distinct purpose. Relying only on the bottom layers gives fast feedback but misses real-world integration issues. Relying only on the top layers gives high confidence but is slow and expensive. A balanced strategy uses all layers.

| Layer | Speed | Cost | Confidence | What It Catches |
|---|---|---|---|---|
| Static Analysis | Seconds | Free | Low-Medium | Syntax errors, misconfigurations, policy violations |
| Plan Validation | Seconds | Free | Medium | Resource changes, unexpected deletes, drift |
| Unit Tests | Seconds | Free | Medium | Logic errors in modules, conditional behavior |
| Integration Tests | Minutes | $$ | High | Real provider behavior, API compatibility |
| End-to-End Tests | Minutes-Hours | $$$ | Very High | Full system behavior, cross-service issues |

---

## Static Analysis

Static analysis tools examine infrastructure code without executing it. They catch syntax errors, style violations, security misconfigurations, and best practice deviations in seconds, making them ideal for the base of the testing pyramid.

### Terraform Validate and Format

Terraform ships with two built-in validation tools that require no additional installation:

```hcl
# terraform validate checks configuration syntax and internal consistency
# It requires terraform init to have been run (for provider schemas)

# Example: Invalid configuration that validate catches
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  nonexistent_attribute = "this will fail validation"  # Error!
}
```

```bash
# Format all Terraform files recursively
terraform fmt -recursive

# Check formatting without modifying files (useful in CI)
terraform fmt -check -recursive

# Validate the current configuration
terraform init -backend=false
terraform validate
```

### TFLint

TFLint is a pluggable linter for Terraform that catches errors that terraform validate cannot — such as invalid instance types, missing required tags, and deprecated resource usage.

```hcl
# .tflint.hcl — TFLint configuration
config {
  call_module_type = "local"
}

plugin "aws" {
  enabled = true
  version = "0.31.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

plugin "azurerm" {
  enabled = true
  version = "0.26.0"
  source  = "github.com/terraform-linters/tflint-ruleset-azurerm"
}

# Custom rules
rule "terraform_naming_convention" {
  enabled = true
  format  = "snake_case"
}

rule "terraform_documented_variables" {
  enabled = true
}

rule "terraform_documented_outputs" {
  enabled = true
}
```

```bash
# Install and run TFLint
tflint --init
tflint --recursive

# Run with specific config file
tflint --config .tflint.hcl

# Output in JSON for CI integration
tflint --format json
```

### Checkov

Checkov is a static analysis tool for infrastructure as code that scans Terraform, CloudFormation, Kubernetes, Helm, ARM templates, and Serverless Framework for security and compliance misconfigurations.

```bash
# Run Checkov against Terraform files
checkov -d . --framework terraform

# Run with specific checks
checkov -d . --check CKV_AWS_18,CKV_AWS_19,CKV_AWS_21

# Skip specific checks
checkov -d . --skip-check CKV_AWS_145

# Output in JUnit format for CI
checkov -d . --output junitxml > checkov-results.xml

# Use a custom config file
checkov -d . --config-file .checkov.yml
```

```yaml
# .checkov.yml — Checkov configuration
soft-fail: false
framework:
  - terraform
skip-check:
  - CKV_AWS_145  # Skip KMS encryption check for dev environments
check:
  - CKV_AWS_18   # Ensure S3 bucket logging is enabled
  - CKV_AWS_19   # Ensure S3 bucket has server-side encryption
  - CKV_AWS_21   # Ensure S3 bucket has versioning enabled
output:
  - cli
  - junitxml
```

### tfsec and Trivy

tfsec (now integrated into Trivy) performs static analysis of Terraform code to detect potential security issues. Trivy is a comprehensive security scanner that covers IaC, containers, and file systems.

```bash
# Run tfsec (standalone)
tfsec .

# Run tfsec with specific severity
tfsec . --minimum-severity HIGH

# Run Trivy for IaC scanning
trivy config .

# Trivy with specific severity filter
trivy config . --severity HIGH,CRITICAL

# Trivy with custom policy directory
trivy config . --policy ./custom-policies
```

```yaml
# .trivyignore — Suppress specific findings
# Ignore specific vulnerability IDs
AVD-AWS-0086
AVD-AWS-0132

# .tfsec.yml — tfsec configuration
minimum_severity: MEDIUM
exclude:
  - aws-s3-enable-bucket-logging
```

### cfn-lint

cfn-lint validates AWS CloudFormation templates against the CloudFormation resource specification and additional rules.

```bash
# Validate a CloudFormation template
cfn-lint template.yaml

# Validate with specific rules
cfn-lint template.yaml --include-checks I

# Validate all templates in a directory
cfn-lint "**/*.yaml" --ignore-templates "**/node_modules/**"

# Output in JSON format
cfn-lint template.yaml --format json
```

---

## Policy as Code

Policy as code frameworks enforce organizational rules — security baselines, compliance requirements, cost controls, and architectural standards — as executable code that runs automatically against infrastructure definitions.

### Open Policy Agent (OPA) with Rego

OPA is a general-purpose policy engine. Rego is its declarative query language. OPA evaluates Terraform plan JSON output against Rego policies to enforce organizational rules.

```rego
# policy/terraform/deny_public_s3.rego
package terraform.s3

import rego.v1

# Deny S3 buckets with public ACLs
deny contains msg if {
    resource := input.resource_changes[_]
    resource.type == "aws_s3_bucket_acl"
    resource.change.after.acl == "public-read"
    msg := sprintf(
        "S3 bucket ACL '%s' uses public-read. All buckets must be private.",
        [resource.address]
    )
}

# Deny S3 buckets without encryption
deny contains msg if {
    resource := input.resource_changes[_]
    resource.type == "aws_s3_bucket"
    not has_encryption(resource.address)
    msg := sprintf(
        "S3 bucket '%s' does not have encryption configured.",
        [resource.address]
    )
}

has_encryption(address) if {
    resource := input.resource_changes[_]
    resource.type == "aws_s3_bucket_server_side_encryption_configuration"
    contains(resource.address, address)
}
```

```rego
# policy/terraform/require_tags.rego
package terraform.tags

import rego.v1

required_tags := {"Environment", "Team", "CostCenter", "ManagedBy"}

# Deny resources missing required tags
deny contains msg if {
    resource := input.resource_changes[_]
    resource.change.after.tags != null
    provided_tags := {key | resource.change.after.tags[key]}
    missing := required_tags - provided_tags
    count(missing) > 0
    msg := sprintf(
        "Resource '%s' is missing required tags: %v",
        [resource.address, missing]
    )
}
```

```bash
# Generate Terraform plan as JSON
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json

# Evaluate OPA policies against the plan
opa eval --data policy/ --input tfplan.json "data.terraform.s3.deny"
opa eval --data policy/ --input tfplan.json "data.terraform.tags.deny"

# Use conftest for a simpler CLI experience
conftest test tfplan.json --policy policy/
```

### HashiCorp Sentinel

Sentinel is HashiCorp's enterprise policy-as-code framework, integrated natively into Terraform Cloud and Terraform Enterprise. Sentinel policies run between the plan and apply phases.

```python
# sentinel/require-instance-tags.sentinel
import "tfplan/v2" as tfplan

required_tags = ["Environment", "Team", "CostCenter"]

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
# sentinel/restrict-instance-types.sentinel
import "tfplan/v2" as tfplan

allowed_instance_types = [
    "t3.micro", "t3.small", "t3.medium",
    "t3.large", "t3.xlarge",
    "m5.large", "m5.xlarge",
]

ec2_instances = filter tfplan.resource_changes as _, rc {
    rc.type is "aws_instance" and
    rc.mode is "managed" and
    (rc.change.actions contains "create" or rc.change.actions contains "update")
}

instance_type_allowed = rule {
    all ec2_instances as _, instance {
        instance.change.after.instance_type in allowed_instance_types
    }
}

main = rule {
    instance_type_allowed
}
```

### AWS Config Rules

AWS Config rules evaluate the compliance of AWS resources against desired configurations. They can be defined as code and deployed alongside infrastructure.

```hcl
# AWS Config rule to ensure S3 bucket encryption
resource "aws_config_config_rule" "s3_encryption" {
  name = "s3-bucket-server-side-encryption-enabled"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_SERVER_SIDE_ENCRYPTION_ENABLED"
  }

  scope {
    compliance_resource_types = ["AWS::S3::Bucket"]
  }
}

# Custom AWS Config rule using Lambda
resource "aws_config_config_rule" "custom_tag_check" {
  name = "required-tags-check"

  source {
    owner             = "CUSTOM_LAMBDA"
    source_identifier = aws_lambda_function.config_rule.arn

    source_detail {
      event_source = "aws.config"
      message_type = "ConfigurationItemChangeNotification"
    }
  }

  input_parameters = jsonencode({
    requiredTags = "Environment,Team,CostCenter"
  })

  depends_on = [aws_config_configuration_recorder.main]
}
```

### Azure Policy

Azure Policy evaluates Azure resources against organizational standards. Policies can be defined in JSON and deployed through Terraform.

```hcl
# Azure Policy definition requiring tags
resource "azurerm_policy_definition" "require_tags" {
  name         = "require-cost-center-tag"
  policy_type  = "Custom"
  mode         = "Indexed"
  display_name = "Require CostCenter tag on all resources"

  metadata = jsonencode({
    category = "Tags"
  })

  policy_rule = jsonencode({
    if = {
      field  = "tags['CostCenter']"
      exists = "false"
    }
    then = {
      effect = "deny"
    }
  })
}

# Assign the policy to a resource group
resource "azurerm_resource_group_policy_assignment" "require_tags" {
  name                 = "require-cost-center-tag"
  resource_group_id    = azurerm_resource_group.main.id
  policy_definition_id = azurerm_policy_definition.require_tags.id
}
```

---

## Plan Validation

Plan validation analyzes the output of `terraform plan` to verify that proposed infrastructure changes match expectations — without actually applying them. This is a low-cost, high-value testing technique.

### Terraform Plan Analysis

```bash
# Generate and inspect a plan
terraform plan -out=tfplan
terraform show tfplan

# Generate plan as JSON for programmatic analysis
terraform show -json tfplan > tfplan.json

# Quick checks: count resources being created, updated, deleted
cat tfplan.json | jq '.resource_changes | group_by(.change.actions[0]) | map({action: .[0].change.actions[0], count: length})'
```

### Plan Assertions

Plan assertions verify specific properties of a Terraform plan using scripts or tools:

```python
#!/usr/bin/env python3
# scripts/validate_plan.py — Assert properties of a Terraform plan

import json
import sys

def load_plan(path):
    with open(path) as f:
        return json.load(f)

def check_no_destroys(plan):
    """Ensure no resources are being destroyed."""
    for change in plan.get("resource_changes", []):
        actions = change["change"]["actions"]
        if "delete" in actions:
            print(f"ERROR: Resource {change['address']} is being destroyed!")
            return False
    return True

def check_no_public_ips(plan):
    """Ensure no EC2 instances have public IPs."""
    for change in plan.get("resource_changes", []):
        if change["type"] == "aws_instance":
            after = change["change"].get("after", {})
            if after.get("associate_public_ip_address"):
                print(f"ERROR: {change['address']} has a public IP!")
                return False
    return True

def check_instance_types(plan, allowed_types):
    """Ensure only allowed instance types are used."""
    for change in plan.get("resource_changes", []):
        if change["type"] == "aws_instance":
            after = change["change"].get("after", {})
            instance_type = after.get("instance_type", "")
            if instance_type not in allowed_types:
                print(f"ERROR: {change['address']} uses {instance_type}!")
                return False
    return True

if __name__ == "__main__":
    plan = load_plan(sys.argv[1])
    allowed = ["t3.micro", "t3.small", "t3.medium", "t3.large"]
    all_passed = True

    if not check_no_destroys(plan):
        all_passed = False
    if not check_no_public_ips(plan):
        all_passed = False
    if not check_instance_types(plan, allowed):
        all_passed = False

    sys.exit(0 if all_passed else 1)
```

### Plan-Based Testing

```bash
# Run plan assertions in CI
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json
python3 scripts/validate_plan.py tfplan.json

# Use conftest for plan-based policy testing
conftest test tfplan.json --policy policy/ --output json
```

### Cost Estimation Validation

Validate cost impact as part of plan review:

```bash
# Use Infracost to estimate cost of planned changes
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json

infracost breakdown --path tfplan.json --format json > cost.json

# Assert cost is within budget
python3 -c "
import json, sys
cost = json.load(open('cost.json'))
monthly = float(cost['totalMonthlyCost'])
if monthly > 500:
    print(f'ERROR: Estimated cost \${monthly}/mo exceeds \$500 budget')
    sys.exit(1)
print(f'OK: Estimated cost \${monthly}/mo is within budget')
"
```

---

## Unit Testing

Unit tests validate the logic of infrastructure modules without deploying real resources. They use mocks or built-in test frameworks to verify that given certain inputs, the module produces expected outputs and resource configurations.

### Terraform Test Framework

Terraform 1.6+ includes a native testing framework using `.tftest.hcl` files:

```hcl
# tests/vpc.tftest.hcl — Test the VPC module

variables {
  vpc_cidr    = "10.0.0.0/16"
  environment = "test"
  name_prefix = "unit-test"
}

run "creates_vpc_with_correct_cidr" {
  command = plan

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block did not match expected value"
  }
}

run "creates_correct_number_of_subnets" {
  command = plan

  assert {
    condition     = length(aws_subnet.private) == 3
    error_message = "Expected 3 private subnets"
  }

  assert {
    condition     = length(aws_subnet.public) == 3
    error_message = "Expected 3 public subnets"
  }
}

run "applies_correct_tags" {
  command = plan

  assert {
    condition     = aws_vpc.main.tags["Environment"] == "test"
    error_message = "Environment tag not set correctly"
  }

  assert {
    condition     = aws_vpc.main.tags["ManagedBy"] == "terraform"
    error_message = "ManagedBy tag not set correctly"
  }
}
```

```hcl
# tests/security_group.tftest.hcl — Test security group module

variables {
  vpc_id      = "vpc-12345678"
  environment = "test"
}

run "denies_ssh_from_anywhere" {
  command = plan

  assert {
    condition = !anytrue([
      for rule in aws_security_group.web.ingress :
      rule.from_port == 22 && contains(rule.cidr_blocks, "0.0.0.0/0")
    ])
    error_message = "Security group must not allow SSH from 0.0.0.0/0"
  }
}

run "allows_https_from_anywhere" {
  command = plan

  assert {
    condition = anytrue([
      for rule in aws_security_group.web.ingress :
      rule.from_port == 443 && contains(rule.cidr_blocks, "0.0.0.0/0")
    ])
    error_message = "Security group must allow HTTPS from anywhere"
  }
}
```

```bash
# Run Terraform tests
terraform test

# Run tests with verbose output
terraform test -verbose

# Run a specific test file
terraform test -filter=tests/vpc.tftest.hcl
```

### Pulumi Unit Tests

Pulumi supports unit testing in general-purpose languages by mocking cloud provider APIs:

```typescript
// __tests__/infrastructure.test.ts — Pulumi unit tests with mocks
import * as pulumi from "@pulumi/pulumi";
import "jest";

// Mock Pulumi runtime before importing infrastructure code
pulumi.runtime.setMocks({
    newResource: function (args: pulumi.runtime.MockResourceArgs): {
        id: string;
        state: Record<string, any>;
    } {
        return {
            id: `${args.name}-id`,
            state: args.inputs,
        };
    },
    call: function (args: pulumi.runtime.MockCallArgs) {
        return args.inputs;
    },
});

describe("S3 Bucket", () => {
    let infra: typeof import("../index");

    beforeAll(async () => {
        infra = await import("../index");
    });

    it("should have versioning enabled", (done) => {
        pulumi.all([infra.bucket.versioning]).apply(([versioning]) => {
            expect(versioning.enabled).toBe(true);
            done();
        });
    });

    it("should have server-side encryption", (done) => {
        pulumi.all([infra.bucket.serverSideEncryptionConfiguration]).apply(
            ([encryption]) => {
                expect(encryption).toBeDefined();
                expect(encryption.rule.applyServerSideEncryptionByDefault.sseAlgorithm)
                    .toBe("aws:kms");
                done();
            }
        );
    });

    it("should block public access", (done) => {
        pulumi.all([infra.bucketPublicAccessBlock]).apply(([block]) => {
            expect(block.blockPublicAcls).toBe(true);
            expect(block.blockPublicPolicy).toBe(true);
            expect(block.ignorePublicAcls).toBe(true);
            expect(block.restrictPublicBuckets).toBe(true);
            done();
        });
    });
});
```

### CDK Assertions

AWS CDK provides assertion libraries for testing synthesized CloudFormation templates:

```python
# test_stack.py — CDK assertion tests in Python
import aws_cdk as cdk
from aws_cdk.assertions import Template, Match
from my_app.vpc_stack import VpcStack

def test_vpc_created_with_correct_cidr():
    app = cdk.App()
    stack = VpcStack(app, "TestVpc", cidr="10.0.0.0/16")
    template = Template.from_stack(stack)

    template.has_resource_properties("AWS::EC2::VPC", {
        "CidrBlock": "10.0.0.0/16",
        "EnableDnsHostnames": True,
        "EnableDnsSupport": True,
    })

def test_creates_private_subnets():
    app = cdk.App()
    stack = VpcStack(app, "TestVpc", cidr="10.0.0.0/16", az_count=3)
    template = Template.from_stack(stack)

    template.resource_count_is("AWS::EC2::Subnet", 6)  # 3 public + 3 private

def test_s3_bucket_encrypted():
    app = cdk.App()
    stack = VpcStack(app, "TestStack")
    template = Template.from_stack(stack)

    template.has_resource_properties("AWS::S3::Bucket", {
        "BucketEncryption": {
            "ServerSideEncryptionConfiguration": [{
                "ServerSideEncryptionByDefault": {
                    "SSEAlgorithm": "aws:kms"
                }
            }]
        }
    })

def test_no_public_s3_buckets():
    app = cdk.App()
    stack = VpcStack(app, "TestStack")
    template = Template.from_stack(stack)

    template.has_resource_properties("AWS::S3::BucketPolicy", {
        "PolicyDocument": Match.object_like({
            "Statement": Match.array_with([
                Match.object_like({
                    "Effect": "Deny",
                    "Principal": "*",
                    "Condition": Match.object_like({
                        "Bool": {"aws:SecureTransport": "false"}
                    })
                })
            ])
        })
    })
```

---

## Integration Testing

Integration tests deploy real infrastructure to a cloud environment, run assertions against the provisioned resources, and then tear everything down. They catch issues that static analysis and unit tests cannot — such as API limitations, IAM permission gaps, and provider bugs.

### Terratest

Terratest is a Go library for writing automated tests that deploy real infrastructure using Terraform, verify the results, and clean up.

```go
// test/vpc_test.go — Terratest integration test for a VPC module
package test

import (
	"fmt"
	"testing"

	"github.com/gruntwork-io/terratest/modules/aws"
	"github.com/gruntwork-io/terratest/modules/random"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

func TestVpcModule(t *testing.T) {
	t.Parallel()

	uniqueID := random.UniqueId()
	awsRegion := "us-west-2"

	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../modules/vpc",
		Vars: map[string]interface{}{
			"vpc_cidr":    "10.0.0.0/16",
			"environment": "test",
			"name_prefix": fmt.Sprintf("test-%s", uniqueID),
		},
		EnvVars: map[string]string{
			"AWS_DEFAULT_REGION": awsRegion,
		},
	})

	// Clean up resources when the test finishes
	defer terraform.Destroy(t, terraformOptions)

	// Deploy the infrastructure
	terraform.InitAndApply(t, terraformOptions)

	// Retrieve outputs
	vpcID := terraform.Output(t, terraformOptions, "vpc_id")
	publicSubnetIDs := terraform.OutputList(t, terraformOptions, "public_subnet_ids")
	privateSubnetIDs := terraform.OutputList(t, terraformOptions, "private_subnet_ids")

	// Verify the VPC exists
	vpc := aws.GetVpcById(t, vpcID, awsRegion)
	assert.Equal(t, "10.0.0.0/16", vpc.CidrBlock)

	// Verify subnet counts
	assert.Equal(t, 3, len(publicSubnetIDs))
	assert.Equal(t, 3, len(privateSubnetIDs))

	// Verify subnets are in different AZs
	publicSubnets := aws.GetSubnetsForVpc(t, vpcID, awsRegion)
	azs := map[string]bool{}
	for _, subnet := range publicSubnets {
		azs[subnet.AvailabilityZone] = true
	}
	require.GreaterOrEqual(t, len(azs), 2, "Subnets should span multiple AZs")
}
```

```go
// test/s3_test.go — Terratest integration test for an S3 module
package test

import (
	"fmt"
	"testing"

	"github.com/gruntwork-io/terratest/modules/aws"
	"github.com/gruntwork-io/terratest/modules/random"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestS3BucketModule(t *testing.T) {
	t.Parallel()

	uniqueID := random.UniqueId()
	awsRegion := "us-west-2"
	bucketName := fmt.Sprintf("test-bucket-%s", uniqueID)

	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../modules/s3",
		Vars: map[string]interface{}{
			"bucket_name": bucketName,
			"environment": "test",
		},
		EnvVars: map[string]string{
			"AWS_DEFAULT_REGION": awsRegion,
		},
	})

	defer terraform.Destroy(t, terraformOptions)
	terraform.InitAndApply(t, terraformOptions)

	// Verify bucket exists
	aws.AssertS3BucketExists(t, awsRegion, bucketName)

	// Verify versioning is enabled
	versioning := aws.GetS3BucketVersioning(t, awsRegion, bucketName)
	assert.Equal(t, "Enabled", versioning)

	// Verify bucket policy blocks public access
	aws.AssertS3BucketPolicyExists(t, awsRegion, bucketName)
}
```

### kitchen-terraform

kitchen-terraform uses Test Kitchen and InSpec to test Terraform configurations:

```yaml
# .kitchen.yml — kitchen-terraform configuration
driver:
  name: terraform
  root_module_directory: modules/vpc
  command_timeout: 600

provisioner:
  name: terraform

verifier:
  name: terraform
  groups:
    - name: default
      controls:
        - vpc_exists
        - subnets_created
        - tags_applied

platforms:
  - name: aws

suites:
  - name: default
    driver:
      variables:
        vpc_cidr: "10.0.0.0/16"
        environment: "test"
```

```ruby
# test/integration/default/controls/vpc_test.rb — InSpec controls

vpc_id = attribute("vpc_id", description: "VPC ID from Terraform output")

control "vpc_exists" do
  impact 1.0
  title "VPC should exist"
  desc "Verify the VPC was created with correct configuration"

  describe aws_vpc(vpc_id) do
    it { should exist }
    its("cidr_block") { should eq "10.0.0.0/16" }
    its("state") { should eq "available" }
    it { should_not be_default }
  end
end

control "subnets_created" do
  impact 1.0
  title "Subnets should exist"

  describe aws_subnets.where(vpc_id: vpc_id) do
    its("count") { should be >= 6 }
  end
end
```

### Test Fixtures and Cleanup

Reliable integration tests require proper fixture management and cleanup to avoid orphaned resources:

```go
// test/fixtures.go — Shared test fixture helpers
package test

import (
	"os"
	"testing"
	"time"

	"github.com/gruntwork-io/terratest/modules/retry"
	"github.com/gruntwork-io/terratest/modules/terraform"
)

// DestroyWithRetry attempts to destroy resources with retries for eventual consistency
func DestroyWithRetry(t *testing.T, opts *terraform.Options) {
	maxRetries := 3
	sleepBetweenRetries := 10 * time.Second

	retry.DoWithRetry(t, "terraform destroy", maxRetries, sleepBetweenRetries,
		func() (string, error) {
			_, err := terraform.DestroyE(t, opts)
			return "", err
		},
	)
}

// SkipIfShort skips integration tests when running in short mode
func SkipIfShort(t *testing.T) {
	if testing.Short() {
		t.Skip("Skipping integration test in short mode")
	}
}

// GetRequiredEnvVar fails the test if an environment variable is not set
func GetRequiredEnvVar(t *testing.T, name string) string {
	value := os.Getenv(name)
	if value == "" {
		t.Fatalf("Required environment variable %s is not set", name)
	}
	return value
}
```

---

## End-to-End Testing

End-to-end tests deploy the complete infrastructure stack — networking, compute, storage, databases, and supporting services — and verify that the entire system works together. These are the most expensive and slowest tests but provide the highest confidence.

### Full Stack Deployment Testing

```go
// test/full_stack_test.go — End-to-end test for complete infrastructure
package test

import (
	"crypto/tls"
	"fmt"
	"net/http"
	"testing"
	"time"

	"github.com/gruntwork-io/terratest/modules/random"
	"github.com/gruntwork-io/terratest/modules/retry"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

func TestFullStackDeployment(t *testing.T) {
	SkipIfShort(t)
	t.Parallel()

	uniqueID := random.UniqueId()

	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../environments/test",
		Vars: map[string]interface{}{
			"environment": fmt.Sprintf("e2e-%s", uniqueID),
		},
	})

	defer DestroyWithRetry(t, terraformOptions)
	terraform.InitAndApply(t, terraformOptions)

	albDNS := terraform.Output(t, terraformOptions, "alb_dns_name")
	dbEndpoint := terraform.Output(t, terraformOptions, "db_endpoint")

	// Verify ALB is reachable
	maxRetries := 10
	sleepBetweenRetries := 30 * time.Second
	url := fmt.Sprintf("https://%s/health", albDNS)

	retry.DoWithRetry(t, "Verify ALB health", maxRetries, sleepBetweenRetries,
		func() (string, error) {
			tlsConfig := &tls.Config{InsecureSkipVerify: true}
			client := &http.Client{
				Transport: &http.Transport{TLSClientConfig: tlsConfig},
				Timeout:   10 * time.Second,
			}
			resp, err := client.Get(url)
			if err != nil {
				return "", err
			}
			defer resp.Body.Close()
			if resp.StatusCode != 200 {
				return "", fmt.Errorf("expected 200, got %d", resp.StatusCode)
			}
			return "", nil
		},
	)

	// Verify database is accessible
	require.NotEmpty(t, dbEndpoint, "Database endpoint should not be empty")
	assert.Contains(t, dbEndpoint, ".rds.amazonaws.com")
}
```

### Smoke Tests

Smoke tests are lightweight end-to-end checks that verify critical paths after deployment:

```bash
#!/bin/bash
# scripts/smoke_test.sh — Post-deployment smoke tests

set -euo pipefail

ALB_DNS="$1"
BASE_URL="https://${ALB_DNS}"

echo "Running smoke tests against ${BASE_URL}"

# Test 1: Health endpoint returns 200
echo -n "Health check... "
STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "${BASE_URL}/health")
if [ "$STATUS" -eq 200 ]; then
    echo "PASS"
else
    echo "FAIL (status: $STATUS)"
    exit 1
fi

# Test 2: API responds to requests
echo -n "API endpoint... "
STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "${BASE_URL}/api/v1/status")
if [ "$STATUS" -eq 200 ]; then
    echo "PASS"
else
    echo "FAIL (status: $STATUS)"
    exit 1
fi

# Test 3: Static assets are served
echo -n "Static assets... "
STATUS=$(curl -sk -o /dev/null -w "%{http_code}" "${BASE_URL}/static/health.html")
if [ "$STATUS" -eq 200 ]; then
    echo "PASS"
else
    echo "FAIL (status: $STATUS)"
    exit 1
fi

echo "All smoke tests passed!"
```

### Cross-Service Validation

Cross-service validation verifies that infrastructure components are correctly integrated:

```go
// test/cross_service_test.go — Verify cross-service connectivity
package test

import (
	"database/sql"
	"fmt"
	"testing"

	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/require"
	_ "github.com/lib/pq"
)

func TestCrossServiceConnectivity(t *testing.T) {
	SkipIfShort(t)

	terraformOptions := &terraform.Options{
		TerraformDir: "../environments/test",
	}

	// Assume infrastructure is already deployed
	dbEndpoint := terraform.Output(t, terraformOptions, "db_endpoint")
	dbPassword := terraform.Output(t, terraformOptions, "db_password")

	// Verify application can connect to database through VPC
	connStr := fmt.Sprintf(
		"host=%s port=5432 user=app password=%s dbname=myapp sslmode=require",
		dbEndpoint, dbPassword,
	)
	db, err := sql.Open("postgres", connStr)
	require.NoError(t, err)
	defer db.Close()

	err = db.Ping()
	require.NoError(t, err, "Application should be able to reach database through VPC")
}
```

---

## Contract Testing

Contract testing verifies that infrastructure modules conform to agreed-upon interfaces — ensuring that module consumers and producers have compatible expectations.

### Module Interface Contracts

Define and test the contract of a Terraform module:

```hcl
# tests/contract.tftest.hcl — Module contract tests

# Contract: the VPC module must always output these values
run "contract_outputs_exist" {
  command = plan

  # The module must output a VPC ID
  assert {
    condition     = output.vpc_id != null
    error_message = "Module must output vpc_id"
  }

  # The module must output subnet IDs as lists
  assert {
    condition     = length(output.private_subnet_ids) > 0
    error_message = "Module must output at least one private subnet ID"
  }

  assert {
    condition     = length(output.public_subnet_ids) > 0
    error_message = "Module must output at least one public subnet ID"
  }
}

# Contract: the module must create resources with required tags
run "contract_tags_applied" {
  command = plan

  assert {
    condition     = aws_vpc.main.tags["ManagedBy"] == "terraform"
    error_message = "All resources must have ManagedBy=terraform tag"
  }
}

# Contract: CIDR block must be a /16
run "contract_cidr_size" {
  command = plan

  variables {
    vpc_cidr = "10.0.0.0/24"
  }

  expect_failures = [
    var.vpc_cidr
  ]
}
```

### Provider Contract Testing

Verify that modules work correctly with specific provider versions:

```hcl
# tests/provider_contract.tftest.hcl

# Ensure the module works with the minimum supported provider version
run "works_with_minimum_provider_version" {
  command = plan

  providers = {
    aws = aws.min_version
  }

  assert {
    condition     = aws_vpc.main.cidr_block == var.vpc_cidr
    error_message = "Module must work with minimum supported AWS provider version"
  }
}
```

### Cross-Team Module Validation

When multiple teams produce and consume shared modules, contract tests prevent breaking changes:

```python
#!/usr/bin/env python3
# scripts/validate_module_contract.py — Validate module interface

import json
import subprocess
import sys

def get_module_outputs(module_path):
    """Extract declared outputs from a Terraform module."""
    result = subprocess.run(
        ["terraform", "providers", "schema", "-json"],
        cwd=module_path, capture_output=True, text=True
    )
    return json.loads(result.stdout)

def validate_contract(module_path, required_outputs):
    """Verify module exposes all required outputs."""
    # Parse outputs.tf to find declared outputs
    with open(f"{module_path}/outputs.tf") as f:
        content = f.read()

    missing = []
    for output in required_outputs:
        if f'output "{output}"' not in content:
            missing.append(output)

    if missing:
        print(f"ERROR: Module missing required outputs: {missing}")
        return False
    print("OK: Module satisfies interface contract")
    return True

if __name__ == "__main__":
    required = ["vpc_id", "private_subnet_ids", "public_subnet_ids", "nat_gateway_ids"]
    ok = validate_contract(sys.argv[1], required)
    sys.exit(0 if ok else 1)
```

---

## Cost Testing

Cost testing validates that infrastructure changes stay within budget constraints. It integrates cost estimation into the development workflow so teams discover cost implications before resources are provisioned.

### Infracost

Infracost estimates the cost of Terraform changes from plan files:

```bash
# Generate cost breakdown from Terraform plan
infracost breakdown --path . --terraform-plan-flags "-var-file=prod.tfvars"

# Compare costs between current and planned state
infracost diff --path . --terraform-plan-flags "-var-file=prod.tfvars"

# Generate cost report as JSON
infracost breakdown --path . --format json --out-file infracost.json

# Post cost estimate as a PR comment
infracost comment github \
  --path infracost.json \
  --repo myorg/myrepo \
  --pull-request 42 \
  --github-token "$GITHUB_TOKEN"
```

```yaml
# infracost.yml — Infracost configuration
version: 0.1
projects:
  - path: environments/production
    name: production
    terraform_plan_flags: "-var-file=prod.tfvars"
  - path: environments/staging
    name: staging
    terraform_plan_flags: "-var-file=staging.tfvars"
```

### Cost Policies

Enforce cost guardrails using Infracost policies:

```rego
# policy/cost/budget.rego — Cost policy with OPA
package infracost

import rego.v1

# Maximum allowed monthly cost increase per PR
max_monthly_increase := 100

# Deny changes that increase monthly cost beyond threshold
deny contains msg if {
    msg := sprintf(
        "Monthly cost increase of $%v exceeds maximum allowed increase of $%v",
        [input.diffTotalMonthlyCost, max_monthly_increase]
    )
    to_number(input.diffTotalMonthlyCost) > max_monthly_increase
}

# Warn on any new resources above a cost threshold
warn contains msg if {
    resource := input.projects[_].breakdown.resources[_]
    to_number(resource.monthlyCost) > 50
    msg := sprintf(
        "Resource '%s' costs $%v/month — review required",
        [resource.name, resource.monthlyCost]
    )
}
```

### Budget Alerts as Code

Define budget alerts as infrastructure to catch cost overruns:

```hcl
# AWS Budget alert defined in Terraform
resource "aws_budgets_budget" "monthly" {
  name         = "monthly-budget"
  budget_type  = "COST"
  limit_amount = "5000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 80
    threshold_type            = "PERCENTAGE"
    notification_type         = "FORECASTED"
    subscriber_email_addresses = ["platform-team@example.com"]
  }

  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 100
    threshold_type            = "PERCENTAGE"
    notification_type         = "ACTUAL"
    subscriber_email_addresses = ["platform-team@example.com", "finance@example.com"]
  }
}

# Azure budget alert defined in Terraform
resource "azurerm_consumption_budget_resource_group" "monthly" {
  name              = "monthly-budget"
  resource_group_id = azurerm_resource_group.main.id
  amount            = 5000
  time_grain        = "Monthly"

  time_period {
    start_date = "2026-01-01T00:00:00Z"
    end_date   = "2026-12-31T00:00:00Z"
  }

  notification {
    enabled   = true
    threshold = 80.0
    operator  = "GreaterThanOrEqualTo"

    contact_emails = ["platform-team@example.com"]
  }
}
```

---

## Security Scanning

Security scanning tools detect misconfigurations, vulnerabilities, and compliance violations in infrastructure code. They complement static analysis by focusing specifically on security concerns.

### Checkov Deep Dive

Checkov supports custom policies and framework-specific scanning:

```yaml
# .checkov.yml — Advanced Checkov configuration
branch: main
compact: true
directory:
  - modules/
  - environments/
framework:
  - terraform
  - terraform_plan
output:
  - cli
  - junitxml
  - sarif
soft-fail: false
skip-check:
  - CKV_AWS_145   # Acceptable for non-production
  - CKV2_AWS_6    # S3 bucket logging handled by org-level config
download-external-modules: true
evaluate-variables: true
```

```python
# custom_checks/require_encryption.py — Custom Checkov policy
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck
from checkov.common.models.enums import CheckCategories, CheckResult

class RequireRDSEncryption(BaseResourceCheck):
    def __init__(self):
        name = "Ensure RDS instances have encryption enabled"
        id = "CUSTOM_AWS_001"
        supported_resources = ["aws_db_instance"]
        categories = [CheckCategories.ENCRYPTION]
        super().__init__(name=name, id=id, categories=categories,
                         supported_resources=supported_resources)

    def scan_resource_conf(self, conf):
        storage_encrypted = conf.get("storage_encrypted", [False])
        if isinstance(storage_encrypted, list):
            storage_encrypted = storage_encrypted[0]
        if storage_encrypted:
            return CheckResult.PASSED
        return CheckResult.FAILED

check = RequireRDSEncryption()
```

### Snyk IaC

Snyk IaC scans Terraform, CloudFormation, Kubernetes, and ARM templates:

```bash
# Scan Terraform files with Snyk
snyk iac test .

# Scan with severity threshold
snyk iac test . --severity-threshold=high

# Output results in SARIF format
snyk iac test . --sarif-file-output=snyk-results.sarif

# Scan with custom rules
snyk iac test . --rules=custom-rules/
```

### KICS

KICS (Keeping Infrastructure as Code Secure) by Checkmarx scans multiple IaC frameworks:

```bash
# Scan all IaC files in the current directory
kics scan -p . -o results/

# Scan with specific queries
kics scan -p . --include-queries "S3 Bucket" -o results/

# Exclude specific queries
kics scan -p . --exclude-queries "Password Without Minimum Length" -o results/

# Output in multiple formats
kics scan -p . --report-formats "json,sarif,html" -o results/
```

### Bridgecrew

Bridgecrew provides a platform layer on top of Checkov with dashboard, drift detection, and supply chain security:

```bash
# Scan with Bridgecrew platform integration
checkov -d . --bc-api-key "$BRIDGECREW_API_KEY" --repo-id myorg/myrepo

# Run with Bridgecrew custom policies from platform
checkov -d . --bc-api-key "$BRIDGECREW_API_KEY" --use-enforcement-rules
```

---

## Testing in CI/CD Pipelines

Integrating IaC tests into CI/CD pipelines ensures every infrastructure change is validated automatically. A well-structured pipeline runs tests in order from fastest to slowest, failing early when possible.

```
IaC CI/CD Pipeline Stages
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  PR Created / Push
       │
       ▼
  ┌──────────┐  < 30s   ┌──────────┐  < 1 min   ┌──────────┐
  │  Format   │─────────▶│  Static  │───────────▶│  Policy  │
  │  + Lint   │          │ Analysis │            │  Check   │
  └──────────┘          └──────────┘            └──────────┘
                                                      │
       ┌──────────────────────────────────────────────┘
       ▼
  ┌──────────┐  < 2 min  ┌──────────┐  < 5 min   ┌──────────┐
  │  Plan    │──────────▶│  Unit    │───────────▶│  Cost    │
  │  + Review│           │  Tests   │            │  Check   │
  └──────────┘          └──────────┘            └──────────┘
                                                      │
       ┌──────────────────────────────────────────────┘
       ▼
  ┌──────────────┐  < 15 min   ┌────────────────┐
  │  Integration │────────────▶│   E2E Tests    │  (on merge)
  │  Tests       │              │   + Smoke      │
  └──────────────┘             └────────────────┘
```

### GitHub Actions Pipeline

```yaml
# .github/workflows/iac-test.yml
name: Infrastructure Tests

on:
  pull_request:
    paths:
      - "infrastructure/**"
      - "modules/**"
  push:
    branches: [main]
    paths:
      - "infrastructure/**"
      - "modules/**"

env:
  TF_VERSION: "1.7.0"
  TF_IN_AUTOMATION: "true"

jobs:
  format-and-lint:
    name: Format & Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: TFLint
        uses: terraform-linters/setup-tflint@v4
        with:
          tflint_version: latest
      - run: |
          tflint --init
          tflint --recursive

  static-analysis:
    name: Static Analysis
    runs-on: ubuntu-latest
    needs: format-and-lint
    steps:
      - uses: actions/checkout@v4

      - name: Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: infrastructure/
          framework: terraform
          output_format: sarif
          output_file_path: checkov-results.sarif

      - name: Trivy IaC Scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: infrastructure/
          severity: HIGH,CRITICAL

      - name: Upload SARIF
        if: always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: checkov-results.sarif

  plan-and-policy:
    name: Plan & Policy Check
    runs-on: ubuntu-latest
    needs: static-analysis
    permissions:
      pull-requests: write
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        working-directory: infrastructure/
        run: terraform init -backend=false

      - name: Terraform Plan
        working-directory: infrastructure/
        run: |
          terraform plan -out=tfplan
          terraform show -json tfplan > tfplan.json

      - name: OPA Policy Check
        uses: open-policy-agent/opa-github-action@v2
        with:
          input: infrastructure/tfplan.json
          policy: policy/

      - name: Infracost
        uses: infracost/actions/setup@v3
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}
      - name: Generate Cost Estimate
        run: |
          infracost breakdown --path infrastructure/tfplan.json \
            --format json --out-file /tmp/infracost.json
          infracost comment github \
            --path /tmp/infracost.json \
            --repo ${{ github.repository }} \
            --pull-request ${{ github.event.pull_request.number }} \
            --github-token ${{ secrets.GITHUB_TOKEN }} \
            --behavior update

  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: static-analysis
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Test
        working-directory: modules/
        run: terraform test -verbose

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: [plan-and-policy, unit-tests]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: "1.22"

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Run Terratest
        working-directory: test/
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: us-west-2
        run: go test -v -timeout 30m ./...
```

### Azure DevOps Pipeline

```yaml
# azure-pipelines.yml — IaC testing pipeline for Azure DevOps
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - infrastructure/*
      - modules/*

pr:
  branches:
    include:
      - main
  paths:
    include:
      - infrastructure/*
      - modules/*

variables:
  TF_VERSION: "1.7.0"
  TF_IN_AUTOMATION: "true"

stages:
  - stage: Validate
    displayName: "Format, Lint & Static Analysis"
    jobs:
      - job: FormatAndLint
        displayName: "Format & Lint"
        pool:
          vmImage: ubuntu-latest
        steps:
          - task: TerraformInstaller@1
            inputs:
              terraformVersion: $(TF_VERSION)

          - script: terraform fmt -check -recursive
            displayName: "Terraform Format Check"

          - script: |
              curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
              tflint --init
              tflint --recursive
            displayName: "TFLint"

      - job: StaticAnalysis
        displayName: "Security Scanning"
        pool:
          vmImage: ubuntu-latest
        steps:
          - script: |
              pip install checkov
              checkov -d infrastructure/ --output junitxml > checkov-results.xml
            displayName: "Checkov Scan"

          - task: PublishTestResults@2
            condition: always()
            inputs:
              testResultsFormat: JUnit
              testResultsFiles: checkov-results.xml

  - stage: Plan
    displayName: "Plan & Policy"
    dependsOn: Validate
    jobs:
      - job: PlanAndPolicy
        displayName: "Terraform Plan & OPA"
        pool:
          vmImage: ubuntu-latest
        steps:
          - task: TerraformInstaller@1
            inputs:
              terraformVersion: $(TF_VERSION)

          - script: |
              cd infrastructure
              terraform init -backend=false
              terraform plan -out=tfplan
              terraform show -json tfplan > tfplan.json
            displayName: "Terraform Plan"

          - script: |
              curl -L -o opa https://openpolicyagent.org/downloads/latest/opa_linux_amd64_static
              chmod +x opa
              ./opa eval --data policy/ --input infrastructure/tfplan.json "data.terraform.deny"
            displayName: "OPA Policy Check"

  - stage: Test
    displayName: "Unit & Integration Tests"
    dependsOn: Plan
    jobs:
      - job: UnitTests
        displayName: "Terraform Test"
        pool:
          vmImage: ubuntu-latest
        steps:
          - task: TerraformInstaller@1
            inputs:
              terraformVersion: $(TF_VERSION)

          - script: |
              cd modules
              terraform test -verbose
            displayName: "Run Terraform Tests"

      - job: IntegrationTests
        displayName: "Terratest"
        dependsOn: UnitTests
        condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
        pool:
          vmImage: ubuntu-latest
        steps:
          - task: GoTool@0
            inputs:
              version: "1.22"

          - script: |
              cd test
              go test -v -timeout 30m ./...
            displayName: "Run Terratest"
            env:
              ARM_CLIENT_ID: $(ARM_CLIENT_ID)
              ARM_CLIENT_SECRET: $(ARM_CLIENT_SECRET)
              ARM_TENANT_ID: $(ARM_TENANT_ID)
              ARM_SUBSCRIPTION_ID: $(ARM_SUBSCRIPTION_ID)
```

### Quality Gates

Define quality gates that must pass before infrastructure changes are applied:

```yaml
# Quality gate configuration (conceptual — enforced in pipeline logic)
quality_gates:
  format:
    required: true
    description: "All files must pass terraform fmt"

  lint:
    required: true
    description: "No TFLint errors"

  static_analysis:
    required: true
    description: "No HIGH or CRITICAL Checkov findings"
    threshold:
      critical: 0
      high: 0

  policy:
    required: true
    description: "All OPA/Sentinel policies pass"

  plan:
    required: true
    description: "Plan succeeds with no unexpected destroys"

  cost:
    required: false
    description: "Cost increase below $500/month"
    soft_fail: true

  unit_tests:
    required: true
    description: "All terraform test files pass"

  integration_tests:
    required: true
    on_branches: [main]
    description: "Terratest suite passes"
```

---

## Test Strategy Comparison

| Test Type | Speed | Cost | Confidence | Complexity | Real Resources | Best For |
|---|---|---|---|---|---|---|
| terraform fmt | Seconds | Free | Low | None | No | Style consistency |
| terraform validate | Seconds | Free | Low | None | No | Syntax errors |
| TFLint | Seconds | Free | Low-Medium | Low | No | Provider-specific errors |
| Checkov / tfsec | Seconds | Free | Medium | Low | No | Security misconfigurations |
| OPA / Sentinel | Seconds | Free | Medium | Medium | No | Organizational policy |
| Plan validation | Seconds | Free | Medium | Medium | No | Change verification |
| Infracost | Seconds | Free tier | Medium | Low | No | Cost estimation |
| Terraform test | Seconds | Free | Medium | Medium | No | Module logic |
| Pulumi unit tests | Seconds | Free | Medium | Medium | No | Module logic (general-purpose lang) |
| CDK assertions | Seconds | Free | Medium | Medium | No | CloudFormation template logic |
| Terratest | Minutes | $$ | High | High | Yes | Real provider behavior |
| kitchen-terraform | Minutes | $$ | High | High | Yes | InSpec-based verification |
| End-to-end tests | Minutes-Hours | $$$ | Very High | Very High | Yes | Full system validation |
| Contract tests | Seconds | Free | Medium | Medium | No | Module interface stability |

```
Choosing the Right Tests
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  "Is this syntactically valid?"         → terraform validate, fmt
  "Does this follow our standards?"      → TFLint, Checkov
  "Does this comply with policy?"        → OPA, Sentinel
  "What will change?"                    → terraform plan
  "Does the module logic work?"          → terraform test, Pulumi unit tests
  "How much will this cost?"             → Infracost
  "Does this work with real providers?"  → Terratest
  "Does the full system work?"           → End-to-end tests
  "Is the module interface stable?"      → Contract tests
```

Recommended test distribution for a mature IaC codebase:

| Test Layer | Recommended Coverage | Run Frequency |
|---|---|---|
| Static analysis + linting | 100% of files | Every commit |
| Policy as code | All security and compliance rules | Every PR |
| Plan validation | Every environment | Every PR |
| Unit tests | All shared modules | Every PR |
| Cost estimation | Production environments | Every PR |
| Integration tests | Critical modules | On merge to main |
| End-to-end tests | Production-like stack | Nightly or weekly |
| Contract tests | All published modules | On module change |

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
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Directory structure, naming, CI/CD integration |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial Testing IaC documentation |
