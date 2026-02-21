# Pulumi

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [What is Pulumi](#what-is-pulumi)
   - [General-Purpose Languages for Infrastructure](#general-purpose-languages-for-infrastructure)
   - [Pulumi Architecture](#pulumi-architecture)
   - [Supported Languages](#supported-languages)
   - [Pulumi Cloud Platform](#pulumi-cloud-platform)
3. [Why Pulumi](#why-pulumi)
   - [Real Languages vs Domain-Specific Languages](#real-languages-vs-domain-specific-languages)
   - [Loops, Conditionals, and Abstractions](#loops-conditionals-and-abstractions)
   - [IDE Support and Developer Experience](#ide-support-and-developer-experience)
   - [Native Testing Capabilities](#native-testing-capabilities)
4. [Getting Started](#getting-started)
   - [Installation](#installation)
   - [Project Structure](#project-structure)
   - [Pulumi.yaml Configuration](#pulumiyaml-configuration)
   - [Stack Configuration](#stack-configuration)
   - [Core CLI Commands](#core-cli-commands)
5. [Core Concepts](#core-concepts)
   - [Resources](#resources)
   - [Component Resources](#component-resources)
   - [Inputs and Outputs](#inputs-and-outputs)
   - [Stack References](#stack-references)
   - [Providers](#providers)
6. [Language Support](#language-support)
   - [TypeScript and JavaScript](#typescript-and-javascript)
   - [Python](#python)
   - [Go](#go)
   - [C# and .NET](#c-and-net)
   - [Java](#java)
7. [Stacks and Environments](#stacks-and-environments)
   - [Dev, Staging, and Production Stacks](#dev-staging-and-production-stacks)
   - [Stack Configuration](#stack-configuration-1)
   - [Stack References for Cross-Stack Dependencies](#stack-references-for-cross-stack-dependencies)
8. [State Management](#state-management)
   - [Pulumi Cloud Backend](#pulumi-cloud-backend)
   - [Self-Managed Backends](#self-managed-backends)
   - [State Backend Comparison](#state-backend-comparison)
9. [Automation API](#automation-api)
   - [Programmatic Infrastructure Management](#programmatic-infrastructure-management)
   - [Embedding Pulumi in Applications](#embedding-pulumi-in-applications)
10. [Pulumi Cloud](#pulumi-cloud)
    - [Console and Dashboard](#console-and-dashboard)
    - [RBAC and Access Controls](#rbac-and-access-controls)
    - [Pulumi Deployments](#pulumi-deployments)
11. [Testing](#testing)
    - [Unit Testing with Mocking](#unit-testing-with-mocking)
    - [Property Testing](#property-testing)
    - [Integration Testing](#integration-testing)
12. [Building Reusable Component Resources](#building-reusable-component-resources)
    - [Component Design Patterns](#component-design-patterns)
    - [Multi-Language Components](#multi-language-components)
13. [Pulumi vs Terraform](#pulumi-vs-terraform)
    - [Feature Comparison](#feature-comparison)
    - [When to Choose Pulumi](#when-to-choose-pulumi)
    - [When to Choose Terraform](#when-to-choose-terraform)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **Pulumi** — an infrastructure as code platform that enables teams to define, deploy, and manage cloud infrastructure using general-purpose programming languages. Unlike DSL tools, Pulumi lets you use TypeScript, Python, Go, C#, and Java with their full ecosystems — IDE support, testing frameworks, package managers, and native language features — to build and manage infrastructure at scale.

### Target Audience

- **Developers** who prefer defining infrastructure using the same languages they use for application code
- **DevOps Engineers** designing and maintaining infrastructure programs, reusable components, and automation pipelines
- **Platform Engineers** building self-service infrastructure platforms with reusable component resources and multi-language support
- **Site Reliability Engineers (SREs)** managing infrastructure state, testing infrastructure code, and ensuring operational consistency across environments
- **Architects** evaluating Pulumi for multi-cloud strategies, organizational standards, and infrastructure management at scale

### Scope

- What Pulumi is, how it works, and how it differs from DSL-based tools like Terraform
- Getting started with the Pulumi CLI, project structure, and stack configuration
- Core concepts: resources, component resources, inputs/outputs, stack references, and providers
- Language support with examples in TypeScript, Python, Go, C#, and Java
- Stack management for multiple environments (dev, staging, production)
- State management: Pulumi Cloud backend vs self-managed backends (S3, Azure Blob, GCS, local)
- Automation API for programmatic infrastructure management
- Pulumi Cloud features: console, RBAC, audit logs, and Deployments
- Testing strategies: unit testing with mocking, property testing, and integration testing

---

## What is Pulumi

### General-Purpose Languages for Infrastructure

Pulumi is an **open-source infrastructure as code platform** that takes a fundamentally different approach from traditional IaC tools. Instead of requiring teams to learn a domain-specific language (DSL) like HCL, Pulumi enables infrastructure definition using **general-purpose programming languages** — TypeScript, Python, Go, C#, and Java.

This approach means that infrastructure code is real code. You can use standard language features (loops, conditionals, functions, classes), existing package managers (npm, pip, Go modules), full IDE capabilities (autocompletion, refactoring, type checking), and native testing frameworks (Jest, pytest, Go testing).

A Pulumi program describes the desired state of your infrastructure. When you run `pulumi up`, the engine computes the minimal set of changes needed:

```typescript
// A complete Pulumi program in TypeScript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const vpc = new aws.ec2.Vpc("main", {
    cidrBlock: "10.0.0.0/16",
    enableDnsHostnames: true,
    tags: { Name: "main-vpc" },
});

// Use a loop to create multiple subnets
const azs = ["us-east-1a", "us-east-1b", "us-east-1c"];
const subnets = azs.map((az, i) =>
    new aws.ec2.Subnet(`subnet-${i}`, {
        vpcId: vpc.id,
        cidrBlock: `10.0.${i}.0/24`,
        availabilityZone: az,
        tags: { Name: `subnet-${az}` },
    })
);

export const vpcId = vpc.id;
export const subnetIds = subnets.map(s => s.id);
```

### Pulumi Architecture

Pulumi uses a **language host** and **deployment engine** architecture that separates your infrastructure program from the execution engine:

```
  Pulumi Architecture
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Your Code                    Pulumi Engine              Cloud Providers
  ┌──────────────────┐         ┌──────────────────┐       ┌──────────────┐
  │  TypeScript /    │         │  Deployment      │       │  AWS         │
  │  Python / Go /   │────────▶│  Engine          │──────▶│  Azure       │
  │  C# / Java      │         │                  │       │  GCP         │
  │                  │         │  ┌────────────┐  │       │  Kubernetes  │
  │  Language Host   │         │  │ Resource   │  │       │  + 100 more  │
  │  (Node/Python/   │         │  │ Providers  │  │       │              │
  │   Go/.NET/JVM)   │         │  └────────────┘  │       └──────────────┘
  └──────────────────┘         └──────────────────┘
          │                            │
          ▼                            ▼
  ┌──────────────────┐         ┌──────────────────┐
  │  npm / pip /     │         │  State Backend   │
  │  go modules /    │         │  (Pulumi Cloud / │
  │  NuGet / Maven   │         │   S3 / Azure /   │
  │  (Dependencies)  │         │   GCS / Local)   │
  └──────────────────┘         └──────────────────┘
```

The architecture works as follows: the **Language Host** runs your program using the native runtime. The **Resource Monitor** receives resource declarations and forwards them to the **Deployment Engine**, which computes a dependency graph, determines the diff, and orchestrates operations. **Resource Providers** communicate with cloud APIs, and the **State Backend** stores the current state.

### Supported Languages

| Language | Runtime | Package Manager | SDK Package |
|---|---|---|---|
| TypeScript / JavaScript | Node.js | npm / yarn | `@pulumi/pulumi` |
| Python | Python 3.8+ | pip / poetry | `pulumi` |
| Go | Go 1.18+ | Go modules | `github.com/pulumi/pulumi/sdk/v3` |
| C# / F# / VB.NET | .NET 6+ | NuGet | `Pulumi` |
| Java / Kotlin / Scala | JVM 11+ | Maven / Gradle | `com.pulumi:pulumi` |

> **Best Practice:** TypeScript is the most widely used Pulumi language and has the broadest ecosystem of examples, components, and community support. If your team is starting with Pulumi, TypeScript is the recommended default.

### Pulumi Cloud Platform

Pulumi Cloud is a managed service that provides state management with automatic locking and versioning, a web-based console, team-based RBAC with SSO/SAML integration, Pulumi Deployments for Git-driven automation, webhooks, audit logs, and built-in secrets encryption. Pulumi Cloud offers a free tier for individual developers and small teams.

---

## Why Pulumi

### Real Languages vs Domain-Specific Languages

The most significant difference between Pulumi and tools like Terraform is the use of **general-purpose programming languages** instead of DSLs. With HCL, you are limited to the constructs provided. With Pulumi, you use the full power of your chosen language:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const config = new pulumi.Config();
const environment = config.require("environment");
const enableMonitoring = config.requireBoolean("enableMonitoring");

// Conditional resource creation with full language support
if (enableMonitoring) {
    const dashboard = new aws.cloudwatch.Dashboard("monitoring", {
        dashboardName: `${environment}-dashboard`,
        dashboardBody: JSON.stringify({
            widgets: buildDashboardWidgets(environment),
        }),
    });
}

// Helper function — not possible in HCL
function buildDashboardWidgets(env: string) {
    const baseWidgets = [
        { type: "metric", properties: { title: "CPU Utilization" } },
        { type: "metric", properties: { title: "Memory Usage" } },
    ];
    if (env === "production") {
        baseWidgets.push(
            { type: "metric", properties: { title: "Error Rate" } },
            { type: "metric", properties: { title: "Latency P99" } },
        );
    }
    return baseWidgets;
}
```

### Loops, Conditionals, and Abstractions

Pulumi programs use standard language constructs for iteration and conditional logic — no special meta-arguments to learn:

```typescript
import * as aws from "@pulumi/aws";

// Standard array operations — no count or for_each meta-arguments
const regions = ["us-east-1", "us-west-2", "eu-west-1"];

const buckets = regions.map(region => {
    const provider = new aws.Provider(`provider-${region}`, { region });
    return new aws.s3.Bucket(`logs-${region}`, {
        bucket: `my-app-logs-${region}`,
        lifecycleRules: [{ enabled: true, expiration: { days: 90 } }],
    }, { provider });
});
```

### IDE Support and Developer Experience

Because Pulumi programs are written in real languages, you get the full IDE experience:

```
  IDE Experience Comparison
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Feature                     Pulumi (TypeScript)    Terraform (HCL)
  ─────────────────────────── ────────────────────── ──────────────────
  Autocompletion               ✅ Full IntelliSense   ⚠️  Limited
  Type checking                ✅ Compile-time        ❌ Runtime only
  Inline documentation         ✅ JSDoc / hover       ⚠️  Limited
  Go-to-definition             ✅ Native              ⚠️  Plugin-based
  Refactoring (rename, etc.)   ✅ Native              ❌ Manual
  Error detection              ✅ Before deployment   ⚠️  terraform validate
  Debugger support             ✅ Full debugger        ❌ Not available
  Code formatting              ✅ Prettier / ESLint   ✅ terraform fmt
```

> **Best Practice:** Use TypeScript with `strict: true` in your `tsconfig.json`. The type system catches configuration errors at compile time — before your infrastructure changes are applied.

### Native Testing Capabilities

One of Pulumi's strongest advantages is the ability to test infrastructure code using standard testing frameworks:

| Testing Level | Purpose | Speed | Cloud Required |
|---|---|---|---|
| Unit testing | Validate resource properties and logic | Fast (seconds) | No |
| Property testing | Assert rules across all resources | Fast (seconds) | No |
| Integration testing | Deploy real resources and verify | Slow (minutes) | Yes |

---

## Getting Started

### Installation

```bash
# Install Pulumi CLI (macOS / Linux)
curl -fsSL https://get.pulumi.com | sh

# Verify installation
pulumi version

# Login to Pulumi Cloud (free tier available)
pulumi login

# Or use a local backend for state storage
pulumi login --local

# Create a new TypeScript project with AWS
pulumi new aws-typescript

# Create a new Python project with Azure
pulumi new azure-python
```

### Project Structure

A typical Pulumi TypeScript project follows this structure:

```
my-pulumi-project/
├── Pulumi.yaml              # Project metadata and runtime configuration
├── Pulumi.dev.yaml           # Stack configuration for 'dev' environment
├── Pulumi.staging.yaml       # Stack configuration for 'staging' environment
├── Pulumi.prod.yaml          # Stack configuration for 'production' environment
├── index.ts                  # Main Pulumi program entry point
├── components/               # Reusable component resources
│   ├── vpc.ts
│   └── database.ts
├── package.json              # Node.js dependencies
├── tsconfig.json             # TypeScript configuration
└── __tests__/                # Unit and property tests
    └── vpc.test.ts
```

### Pulumi.yaml Configuration

The `Pulumi.yaml` file defines the project metadata:

```yaml
# Pulumi.yaml — project configuration
name: my-infrastructure
description: Production cloud infrastructure
runtime:
  name: nodejs
  options:
    typescript: true

config:
  aws:region:
    default: us-east-1
```

For Python projects:

```yaml
name: my-infrastructure
runtime:
  name: python
  options:
    virtualenv: venv
```

### Stack Configuration

Stack configuration files (`Pulumi.<stack>.yaml`) hold per-environment values:

```yaml
# Pulumi.dev.yaml
config:
  aws:region: us-east-1
  my-infrastructure:environment: dev
  my-infrastructure:instanceType: t3.micro
  my-infrastructure:minNodes: "1"
  my-infrastructure:maxNodes: "3"
  my-infrastructure:dbPassword:
    secure: AAABADQXFlU0mxol3gFTSFxs+oU6cuqYKLjzVMBAhD0PDuY=
```

Access configuration values in your program:

```typescript
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config();

// Required values — program fails if not set
const environment = config.require("environment");
const dbPassword = config.requireSecret("dbPassword");

// Optional values with defaults
const instanceType = config.get("instanceType") || "t3.micro";
const minNodes = config.getNumber("minNodes") || 1;
```

### Core CLI Commands

```bash
# Create a new stack
pulumi stack init dev

# Set configuration values
pulumi config set environment dev
pulumi config set --secret dbPassword my-secret-password

# Preview changes (dry run)
pulumi preview

# Deploy infrastructure
pulumi up

# Deploy with automatic approval (CI/CD)
pulumi up --yes

# View stack outputs
pulumi stack output

# Destroy all resources in the stack
pulumi destroy

# Remove the stack entirely
pulumi stack rm dev
```

```
  Pulumi CLI Workflow
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Developer                  Pulumi CLI                 Cloud Provider
  ┌──────────┐               ┌─────────────┐            ┌─────────────┐
  │  Write   │──pulumi up──▶│  1. Run     │            │             │
  │  Code    │               │     program │            │             │
  │          │◀──preview────│  2. Show    │            │             │
  │  Review  │               │     plan    │            │             │
  │  & Confirm│──yes────────▶│  3. Apply  │───────────▶│  Create /   │
  │          │               │     changes │            │  Update /   │
  │          │◀──outputs────│  4. Update │            │  Delete     │
  │          │               │     state   │◀───────────│  Resources  │
  └──────────┘               └─────────────┘            └─────────────┘
```

---

## Core Concepts

### Resources

Resources are the fundamental building blocks in Pulumi. Each resource represents a single cloud infrastructure object:

```typescript
import * as aws from "@pulumi/aws";

// Create a resource: new <ResourceType>(<name>, <args>, <options>)
const bucket = new aws.s3.Bucket("my-bucket", {
    bucket: "my-app-assets-bucket",
    versioning: { enabled: true },
    serverSideEncryptionConfiguration: {
        rule: {
            applyServerSideEncryptionByDefault: { sseAlgorithm: "aws:kms" },
        },
    },
    tags: { Environment: "production", ManagedBy: "pulumi" },
});

// Resource options control behavior
const instance = new aws.ec2.Instance("web-server", {
    ami: "ami-0c55b159cbfafe1f0",
    instanceType: "t3.micro",
}, {
    protect: true,              // Prevent accidental deletion
    ignoreChanges: ["tags"],    // Ignore changes to specific properties
    dependsOn: [bucket],        // Explicit dependency
});
```

> **Best Practice:** Use the `protect` resource option on critical production resources (databases, DNS zones, encryption keys) to prevent accidental deletion during `pulumi destroy` or refactoring.

### Component Resources

Component resources are logical groupings of related resources that form a single, reusable abstraction:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

interface VpcNetworkArgs {
    cidrBlock: string;
    cidrPrefix: string;
    availabilityZones: string[];
}

class VpcNetwork extends pulumi.ComponentResource {
    public readonly vpcId: pulumi.Output<string>;
    public readonly subnetIds: pulumi.Output<string>[];

    constructor(name: string, args: VpcNetworkArgs, opts?: pulumi.ComponentResourceOptions) {
        super("custom:networking:VpcNetwork", name, {}, opts);

        const vpc = new aws.ec2.Vpc(`${name}-vpc`, {
            cidrBlock: args.cidrBlock,
            enableDnsHostnames: true,
            tags: { Name: `${name}-vpc` },
        }, { parent: this });

        this.vpcId = vpc.id;
        this.subnetIds = args.availabilityZones.map((az, i) => {
            const subnet = new aws.ec2.Subnet(`${name}-subnet-${i}`, {
                vpcId: vpc.id,
                cidrBlock: `${args.cidrPrefix}.${i}.0/24`,
                availabilityZone: az,
                tags: { Name: `${name}-${az}` },
            }, { parent: this });
            return subnet.id;
        });

        this.registerOutputs({ vpcId: this.vpcId, subnetIds: this.subnetIds });
    }
}

// Usage — a single line creates VPC + subnets
const network = new VpcNetwork("production", {
    cidrBlock: "10.0.0.0/16",
    cidrPrefix: "10.0",
    availabilityZones: ["us-east-1a", "us-east-1b", "us-east-1c"],
});
```

### Inputs and Outputs

Pulumi uses an **Input/Output** system to handle values not known until deployment time:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

const bucket = new aws.s3.Bucket("data", {});

// bucket.id is an Output<string> — not a plain string
// Use .apply() to transform output values
const bucketUrl = bucket.bucket.apply(name => `https://${name}.s3.amazonaws.com`);

// Use pulumi.interpolate for string interpolation with outputs
const bucketEndpoint = pulumi.interpolate`https://${bucket.bucket}.s3.amazonaws.com`;

// Use pulumi.all() to combine multiple outputs
const summary = pulumi.all([bucket.id, bucket.arn]).apply(
    ([id, arn]) => `Bucket ${id} has ARN ${arn}`
);

// Pass outputs directly to other resources — Pulumi resolves them automatically
const policy = new aws.s3.BucketPolicy("data-policy", {
    bucket: bucket.id,
    policy: bucket.arn.apply(arn => JSON.stringify({
        Version: "2012-10-17",
        Statement: [{
            Effect: "Allow",
            Principal: "*",
            Action: "s3:GetObject",
            Resource: `${arn}/*`,
        }],
    })),
});
```

> **Best Practice:** Prefer `pulumi.interpolate` over `.apply()` for simple string concatenation. Use `.apply()` when you need to transform values with complex logic.

### Stack References

Stack references allow one stack to read outputs from another stack:

```typescript
// In the networking stack — export outputs
export const vpcId = vpc.id;
export const privateSubnetIds = subnets.map(s => s.id);

// In the application stack — consume outputs
import * as pulumi from "@pulumi/pulumi";

const networkStack = new pulumi.StackReference("myorg/networking/production");
const vpcId = networkStack.getOutput("vpcId");
const subnetIds = networkStack.getOutput("privateSubnetIds");
```

### Providers

Providers communicate with cloud APIs. Configure multiple providers for multi-region or multi-account deployments:

```typescript
import * as aws from "@pulumi/aws";

// Default provider — uses stack configuration
const defaultBucket = new aws.s3.Bucket("default-bucket", {});

// Explicit provider for a different region
const euProvider = new aws.Provider("eu-west-1", { region: "eu-west-1" });

const euBucket = new aws.s3.Bucket("eu-bucket", {
    bucket: "my-app-eu-assets",
}, { provider: euProvider });

// Explicit provider for a different AWS account
const stagingProvider = new aws.Provider("staging", {
    region: "us-east-1",
    assumeRole: { roleArn: "arn:aws:iam::123456789012:role/PulumiDeployRole" },
});
```

---

## Language Support

### TypeScript and JavaScript

TypeScript is the most popular Pulumi language, offering strong typing and broad ecosystem support:

```typescript
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";

const vpc = new awsx.ec2.Vpc("app-vpc", {
    numberOfAvailabilityZones: 3,
    natGateways: { strategy: awsx.ec2.NatGatewayStrategy.Single },
});

const cluster = new aws.ecs.Cluster("app-cluster", {});

const service = new awsx.ecs.FargateService("app-service", {
    cluster: cluster.arn,
    networkConfiguration: { subnets: vpc.privateSubnetIds, securityGroups: [] },
    desiredCount: 3,
    taskDefinitionArgs: {
        container: {
            name: "app",
            image: "nginx:latest",
            cpu: 256,
            memory: 512,
            essential: true,
            portMappings: [{ containerPort: 80 }],
        },
    },
});
```

### Python

Python support makes Pulumi accessible to data engineering and ML platform teams:

```python
import pulumi
import pulumi_aws as aws

config = pulumi.Config()
environment = config.require("environment")

vpc = aws.ec2.Vpc("main",
    cidr_block="10.0.0.0/16",
    enable_dns_hostnames=True,
    tags={"Name": f"main-{environment}"})

availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
subnets = [
    aws.ec2.Subnet(f"subnet-{i}",
        vpc_id=vpc.id,
        cidr_block=f"10.0.{i}.0/24",
        availability_zone=az,
        tags={"Name": f"subnet-{az}"})
    for i, az in enumerate(availability_zones)
]

db = aws.rds.Instance("database",
    engine="postgres",
    instance_class="db.t3.micro",
    allocated_storage=20,
    db_name="myapp",
    username="admin",
    password=config.require_secret("dbPassword"),
    skip_final_snapshot=environment == "dev",
    tags={"Environment": environment})

pulumi.export("vpc_id", vpc.id)
pulumi.export("db_endpoint", db.endpoint)
```

### Go

Go support appeals to teams building Kubernetes operators and cloud-native infrastructure:

```go
package main

import (
    "github.com/pulumi/pulumi-aws/sdk/v6/go/aws/ec2"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi/config"
)

func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        cfg := config.New(ctx, "")
        environment := cfg.Require("environment")

        vpc, err := ec2.NewVpc(ctx, "main", &ec2.VpcArgs{
            CidrBlock:          pulumi.String("10.0.0.0/16"),
            EnableDnsHostnames: pulumi.Bool(true),
            Tags: pulumi.StringMap{
                "Name": pulumi.Sprintf("main-%s", environment),
            },
        })
        if err != nil {
            return err
        }

        ctx.Export("vpcId", vpc.ID())
        return nil
    })
}
```

### C# and .NET

```csharp
using Pulumi;
using Pulumi.Aws.Ec2;
using System.Collections.Generic;

return await Deployment.RunAsync(() =>
{
    var config = new Config();
    var environment = config.Require("environment");

    var vpc = new Vpc("main", new VpcArgs
    {
        CidrBlock = "10.0.0.0/16",
        EnableDnsHostnames = true,
        Tags = new InputMap<string> { { "Name", $"main-{environment}" } },
    });

    return new Dictionary<string, object?> { ["vpcId"] = vpc.Id };
});
```

### Java

```java
package myinfra;

import com.pulumi.Pulumi;
import com.pulumi.aws.ec2.Vpc;
import com.pulumi.aws.ec2.VpcArgs;
import java.util.Map;

public class App {
    public static void main(String[] args) {
        Pulumi.run(ctx -> {
            var config = ctx.config();
            var environment = config.require("environment");

            var vpc = new Vpc("main", VpcArgs.builder()
                .cidrBlock("10.0.0.0/16")
                .enableDnsHostnames(true)
                .tags(Map.of("Name", String.format("main-%s", environment)))
                .build());

            ctx.export("vpcId", vpc.id());
        });
    }
}
```

---

## Stacks and Environments

### Dev, Staging, and Production Stacks

Pulumi uses **stacks** to represent isolated instances of the same infrastructure program. Each stack has its own configuration and state:

```
  Stack Architecture
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Pulumi Program (index.ts)
  ┌──────────────────────────────────────────────────────────────┐
  │   const config = new pulumi.Config();                       │
  │   const env = config.require("environment");                │
  │   // ... same code for all environments                     │
  └──────────────┬───────────────┬───────────────┬──────────────┘
                 │               │               │
                 ▼               ▼               ▼
  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
  │  dev stack       │ │  staging stack   │ │  prod stack      │
  │  Pulumi.dev.yaml │ │  Pulumi.stg.yaml│ │  Pulumi.prod.yaml│
  │  t3.micro        │ │  t3.medium       │ │  t3.xlarge       │
  │  1 AZ            │ │  2 AZs           │ │  3 AZs           │
  └──────────────────┘ └──────────────────┘ └──────────────────┘
```

```bash
# Create stacks for each environment
pulumi stack init dev
pulumi stack init staging
pulumi stack init prod

# Switch between stacks
pulumi stack select dev

# List all stacks
pulumi stack ls
```

### Stack Configuration

Each stack has its own configuration file:

```yaml
# Pulumi.prod.yaml
config:
  aws:region: us-east-1
  my-app:environment: production
  my-app:instanceType: t3.xlarge
  my-app:enableMultiAz: "true"
  my-app:dbPassword:
    secure: AAABAJk7qB2sN0jJ29xTz5mKF8gHlP0xR4kT9sA1bC2dE3=
```

Use configuration to adapt behavior per environment:

### Stack References for Cross-Stack Dependencies

Stack references enable microservice-style infrastructure where independent teams manage separate stacks:

```
  Cross-Stack Reference Flow
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Networking Stack                Application Stack
  ┌────────────────────┐          ┌────────────────────┐
  │  VPC, Subnets,     │          │  ECS Service,      │
  │  NAT Gateways      │          │  Load Balancer     │
  │  Exports:          │─────────▶│  References:        │
  │   vpcId            │          │   vpcId            │
  │   privateSubnetIds │          │   privateSubnetIds │
  └────────────────────┘          └────────────────────┘
```

---

## State Management

### Pulumi Cloud Backend

The **default and recommended** state backend is Pulumi Cloud, which provides encryption at rest and in transit, automatic concurrency locking, state versioning with full history, audit logs, and per-stack encryption keys for secrets.

```bash
# Login to Pulumi Cloud (default)
pulumi login

# Login with an access token (CI/CD)
export PULUMI_ACCESS_TOKEN=pul-abc123...
pulumi login
```

### Self-Managed Backends

For organizations that cannot use Pulumi Cloud, self-managed backends store state in cloud object storage or locally:

```bash
# Amazon S3 backend
pulumi login s3://my-pulumi-state-bucket

# Azure Blob Storage backend
pulumi login azblob://my-pulumi-state-container

# Google Cloud Storage backend
pulumi login gs://my-pulumi-state-bucket

# Local filesystem backend
pulumi login --local
```

> **Best Practice:** When using self-managed backends, you must configure your own locking, encryption, and access control. Always enable versioning on your state bucket.

### State Backend Comparison

| Feature | Pulumi Cloud | S3 | Azure Blob | GCS | Local |
|---|---|---|---|---|---|
| Encryption at rest | ✅ Built-in | ✅ SSE/KMS | ✅ SSE | ✅ SSE | ❌ |
| Automatic locking | ✅ Built-in | ⚠️ DynamoDB | ⚠️ Lease | ⚠️ Lock file | ❌ |
| State versioning | ✅ Built-in | ✅ S3 versioning | ✅ Blob versioning | ✅ Object versioning | ❌ |
| Web console | ✅ | ❌ | ❌ | ❌ | ❌ |
| Secrets encryption | ✅ Per-stack | ⚠️ KMS | ⚠️ Key Vault | ⚠️ Cloud KMS | ⚠️ Passphrase |
| Audit logs | ✅ Built-in | ⚠️ CloudTrail | ⚠️ Monitor | ⚠️ Cloud Audit | ❌ |
| Team collaboration | ✅ Native | ⚠️ Manual | ⚠️ Manual | ⚠️ Manual | ❌ |
| Setup complexity | Low | Medium | Medium | Medium | None |

---

## Automation API

### Programmatic Infrastructure Management

The **Automation API** is a programmatic interface to the Pulumi engine that lets you embed infrastructure management directly in your application code:

```typescript
import { LocalWorkspace } from "@pulumi/pulumi/automation";

async function deployInfrastructure() {
    const pulumiProgram = async () => {
        const aws = await import("@pulumi/aws");
        const bucket = new aws.s3.Bucket("auto-bucket", {
            bucket: "my-automation-bucket",
            tags: { ManagedBy: "automation-api" },
        });
        return { bucketName: bucket.bucket };
    };

    const stack = await LocalWorkspace.createOrSelectStack({
        stackName: "dev",
        projectName: "automation-demo",
        program: pulumiProgram,
    });

    await stack.setConfig("aws:region", { value: "us-east-1" });

    // Preview and deploy
    const preview = await stack.preview();
    console.log("Preview:", preview.changeSummary);

    const result = await stack.up({ onOutput: console.log });
    console.log("Bucket:", result.outputs.bucketName.value);
}
```

### Embedding Pulumi in Applications

The Automation API enables powerful patterns for building infrastructure platforms:

```
  Automation API Architecture
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Your Application (REST API / CLI / CI Pipeline)
  ┌──────────────────────────────────────────────────────────────┐
  │  ┌─────────────┐  ┌────────────┐  ┌──────────────┐         │
  │  │ Workspace   │  │ Stack      │  │ Program      │         │
  │  │ Management  │  │ Operations │  │ Definition   │         │
  │  └─────────────┘  └────────────┘  └──────────────┘         │
  └──────────────────────────┬──────────────────────────────────┘
                             ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  Pulumi Engine → Resource Providers → Cloud APIs            │
  └──────────────────────────────────────────────────────────────┘
```

Common patterns built with the Automation API:

| Pattern | Description |
|---|---|
| Self-service platforms | REST APIs that provision infrastructure on demand |
| CI/CD integration | Embed Pulumi in custom pipelines without the CLI |
| Multi-tenant SaaS | Per-tenant infrastructure provisioning |
| Testing frameworks | Programmatic stack creation for integration tests |
| Drift detection | Scheduled programs that detect and remediate drift |
| GitOps controllers | Kubernetes operators that reconcile infrastructure |

---

## Pulumi Cloud

### Console and Dashboard

The Pulumi Cloud console provides a web-based interface for managing infrastructure:

```
  Pulumi Cloud Console
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌──────────────────────────────────────────────────────────────┐
  │  Organization: myorg                                        │
  │                                                              │
  │  ┌──────────────┬──────────┬──────────────┬───────────────┐  │
  │  │ Stack        │ Status   │ Last Updated │ Resources     │  │
  │  ├──────────────┼──────────┼──────────────┼───────────────┤  │
  │  │ net/prod     │ ✅ OK    │ 2h ago       │ 24            │  │
  │  │ app/prod     │ ✅ OK    │ 1h ago       │ 47            │  │
  │  │ app/staging  │ ⚠️ Drift │ 3h ago       │ 42            │  │
  │  └──────────────┴──────────┴──────────────┴───────────────┘  │
  └──────────────────────────────────────────────────────────────┘
```

The console includes stack overview, resource explorer, activity feed, and a visual dependency graph.

### RBAC and Access Controls

Pulumi Cloud provides role-based access control:

| Role | Permissions |
|---|---|
| Admin | Full access: manage members, stacks, settings, billing |
| Write | Create, update, and destroy stacks and resources |
| Read | View stacks, resources, configuration, and activity |

Access can be scoped to organization, team, or individual stack level. Pulumi Cloud records audit logs of every action for compliance and troubleshooting, which can be exported to external SIEM systems.

> **Best Practice:** Follow the principle of least privilege. Grant `Read` access by default and `Write` access only to teams that need to deploy. Restrict `Admin` to platform and security teams.

### Pulumi Deployments

Pulumi Deployments is a managed deployment service built into Pulumi Cloud that provides Git push-to-deploy, review stacks for pull request review, scheduled drift detection, TTL stacks (auto-destroy after a duration), and deployment triggers via webhooks, schedules, or API calls.

---

## Testing

### Unit Testing with Mocking

Pulumi supports unit testing by mocking the Pulumi engine. Tests run without cloud credentials and execute in seconds:

```typescript
// __tests__/infrastructure.test.ts
import * as pulumi from "@pulumi/pulumi";

pulumi.runtime.setMocks({
    newResource(args: pulumi.runtime.MockResourceArgs): {
        id: string;
        state: Record<string, any>;
    } {
        return {
            id: `${args.name}-mock-id`,
            state: {
                ...args.inputs,
                arn: `arn:aws:s3:::${args.inputs.bucket || args.name}`,
            },
        };
    },
    call(args: pulumi.runtime.MockCallArgs) {
        return args.inputs;
    },
});

describe("S3 Bucket Infrastructure", () => {
    let infra: typeof import("../index");

    beforeAll(async () => {
        infra = await import("../index");
    });

    test("bucket has versioning enabled", async () => {
        const versioning = await new Promise<any>((resolve) =>
            infra.dataBucket.versioning.apply(v => resolve(v))
        );
        expect(versioning.enabled).toBe(true);
    });

    test("bucket name follows naming convention", async () => {
        const name = await new Promise<string>((resolve) =>
            infra.dataBucket.bucket.apply(n => resolve(n))
        );
        expect(name).toMatch(/^my-app-data-/);
    });
});
```

```bash
# Run unit tests with Jest
npx jest

# Run with coverage
npx jest --coverage
```

### Property Testing

Property tests validate rules across **all** resources in a stack:

```typescript
// __tests__/policies.test.ts
import * as pulumi from "@pulumi/pulumi";

let resources: any[] = [];

pulumi.runtime.setMocks({
    newResource(args: pulumi.runtime.MockResourceArgs) {
        resources.push({ type: args.type, name: args.name, inputs: args.inputs });
        return { id: `${args.name}-id`, state: args.inputs };
    },
    call(args: pulumi.runtime.MockCallArgs) {
        return args.inputs;
    },
});

describe("Infrastructure Policies", () => {
    beforeAll(async () => { await import("../index"); });

    test("all S3 buckets have encryption enabled", () => {
        const buckets = resources.filter(r => r.type === "aws:s3/bucket:Bucket");
        for (const bucket of buckets) {
            expect(bucket.inputs.serverSideEncryptionConfiguration).toBeDefined();
        }
    });

    test("no security groups allow ingress from 0.0.0.0/0", () => {
        const sgs = resources.filter(
            r => r.type === "aws:ec2/securityGroup:SecurityGroup"
        );
        for (const sg of sgs) {
            for (const rule of (sg.inputs.ingress || [])) {
                expect(rule.cidrBlocks).not.toContain("0.0.0.0/0");
            }
        }
    });
});
```

### Integration Testing

Integration tests deploy real infrastructure, validate it, and tear it down. These require cloud credentials and take minutes to run:

```typescript
// __tests__/integration.test.ts
import { LocalWorkspace, Stack } from "@pulumi/pulumi/automation";

describe("Full Stack Integration", () => {
    let stack: Stack;
    const stackName = `test-${Date.now()}`;

    beforeAll(async () => {
        stack = await LocalWorkspace.createOrSelectStack({
            stackName,
            projectName: "integration-test",
            program: async () => {
                const aws = await import("@pulumi/aws");
                const bucket = new aws.s3.Bucket("test-bucket", {
                    bucket: `integration-test-${stackName}`,
                    forceDestroy: true,
                });
                return { bucketName: bucket.bucket };
            },
        });
        await stack.setConfig("aws:region", { value: "us-east-1" });
        await stack.up({ onOutput: console.log });
    }, 300_000);

    afterAll(async () => {
        await stack.destroy({ onOutput: console.log });
        await stack.workspace.removeStack(stackName);
    }, 300_000);

    test("bucket was created", async () => {
        const outputs = await stack.outputs();
        expect(outputs.bucketName.value).toContain("integration-test");
    });
});
```

> **Best Practice:** Run unit and property tests on every pull request. Run integration tests on a schedule (nightly or weekly) or before major releases. Use ephemeral stack names with timestamps to prevent test collisions.

---

## Building Reusable Component Resources

### Component Design Patterns

Component resources let you build reusable, self-contained infrastructure modules shared across projects:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";

export interface StaticWebsiteArgs {
    domain: string;
    certificateArn: pulumi.Input<string>;
    indexDocument?: string;
    tags?: Record<string, string>;
}

export class StaticWebsite extends pulumi.ComponentResource {
    public readonly bucketName: pulumi.Output<string>;
    public readonly distributionId: pulumi.Output<string>;
    public readonly websiteUrl: pulumi.Output<string>;

    constructor(name: string, args: StaticWebsiteArgs, opts?: pulumi.ComponentResourceOptions) {
        super("custom:web:StaticWebsite", name, {}, opts);
        const parentOpts = { parent: this };

        const bucket = new aws.s3.Bucket(`${name}-content`, {
            bucket: args.domain,
            website: {
                indexDocument: args.indexDocument || "index.html",
                errorDocument: "error.html",
            },
            tags: args.tags,
        }, parentOpts);

        const oai = new aws.cloudfront.OriginAccessIdentity(`${name}-oai`, {
            comment: `OAI for ${args.domain}`,
        }, parentOpts);

        new aws.s3.BucketPolicy(`${name}-policy`, {
            bucket: bucket.id,
            policy: pulumi.all([bucket.arn, oai.iamArn]).apply(
                ([arn, oaiArn]) => JSON.stringify({
                    Version: "2012-10-17",
                    Statement: [{
                        Effect: "Allow",
                        Principal: { AWS: oaiArn },
                        Action: "s3:GetObject",
                        Resource: `${arn}/*`,
                    }],
                })
            ),
        }, parentOpts);

        const cdn = new aws.cloudfront.Distribution(`${name}-cdn`, {
            enabled: true,
            aliases: [args.domain],
            origins: [{
                domainName: bucket.bucketRegionalDomainName,
                originId: "s3Origin",
                s3OriginConfig: {
                    originAccessIdentity: oai.cloudfrontAccessIdentityPath,
                },
            }],
            defaultCacheBehavior: {
                allowedMethods: ["GET", "HEAD"],
                cachedMethods: ["GET", "HEAD"],
                targetOriginId: "s3Origin",
                forwardedValues: { queryString: false, cookies: { forward: "none" } },
                viewerProtocolPolicy: "redirect-to-https",
            },
            restrictions: { geoRestriction: { restrictionType: "none" } },
            viewerCertificate: {
                acmCertificateArn: args.certificateArn,
                sslSupportMethod: "sni-only",
            },
        }, parentOpts);

        this.bucketName = bucket.bucket;
        this.distributionId = cdn.id;
        this.websiteUrl = pulumi.interpolate`https://${args.domain}`;
        this.registerOutputs({
            bucketName: this.bucketName,
            distributionId: this.distributionId,
            websiteUrl: this.websiteUrl,
        });
    }
}

// Usage — a single line deploys S3 + CloudFront + policy
const site = new StaticWebsite("marketing", {
    domain: "marketing.example.com",
    certificateArn: "arn:aws:acm:us-east-1:123456789012:certificate/abc-123",
});
```

### Multi-Language Components

Pulumi allows you to author a component in one language and consume it from any other using **multi-language components** (MLCs). A `schema.yaml` defines the component's inputs, outputs, and structure, from which SDKs are generated for all supported languages:

```
  Multi-Language Component Architecture
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Component Author                          Component Consumers
  ┌────────────────────────┐                ┌─────────────────────┐
  │  TypeScript Component  │                │  TypeScript program │
  │  ┌──────────────────┐  │    Schema      │  import { Vpc }     │
  │  │ VpcNetwork class │  │───────────────▶├─────────────────────┤
  │  │   - VPC          │  │    (generates  │  Python program     │
  │  │   - Subnets      │  │     SDKs for   │  from myorg import  │
  │  └──────────────────┘  │     all langs) ├─────────────────────┤
  │                        │               │  Go / C# / Java     │
  └────────────────────────┘               └─────────────────────┘
```

Multi-language components are published to standard registries (npm, PyPI, Go modules, NuGet, Maven Central).

---

## Pulumi vs Terraform

### Feature Comparison

| Feature | Pulumi | Terraform |
|---|---|---|
| **Language** | TypeScript, Python, Go, C#, Java | HCL (domain-specific) |
| **Language features** | Full: loops, classes, functions, generics | Limited: count, for_each, dynamic blocks |
| **IDE support** | Full IntelliSense, refactoring, debugging | Limited (plugin-based) |
| **Type safety** | Compile-time type checking | Runtime validation only |
| **Testing** | Native unit, property, and integration tests | Limited: `terraform test` (v1.6+) |
| **Package management** | npm, pip, Go modules, NuGet, Maven | Terraform Registry modules |
| **Secrets** | Built-in per-stack encryption | Sensitive variables (not encrypted in state) |
| **Providers** | 100+ (many bridged from Terraform) | 3,000+ |
| **Module ecosystem** | Growing (Pulumi Registry) | Mature (Terraform Registry) |
| **Automation API** | Full programmatic API | Limited (CLI wrappers, cdktf) |
| **Policy as code** | CrossGuard (TypeScript, Python, OPA) | Sentinel (Enterprise), OPA |
| **Drift detection** | Pulumi Deployments, `pulumi refresh` | `terraform plan` |
| **License** | Apache 2.0 | BSL 1.1 (was MPL 2.0) |

### When to Choose Pulumi

- **Your team already knows TypeScript, Python, Go, C#, or Java** and does not want to learn HCL
- **You need complex logic** — loops, conditionals, string manipulation, dynamic generation
- **Testing is a priority** — unit test infrastructure with standard testing frameworks
- **You are building a platform** — Automation API enables self-service provisioning
- **Secrets management matters** — Pulumi encrypts secrets in state by default

### When to Choose Terraform

- **Your team already knows HCL** and has existing Terraform codebases
- **You need the broadest provider ecosystem** — Terraform has 3,000+ providers vs Pulumi's 100+
- **You prefer a DSL** — HCL's declarative-only syntax prevents imperative anti-patterns
- **Community and hiring** — larger community, more tutorials, and more practitioners

---

## Next Steps

Continue your Infrastructure as Code learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | IaC Fundamentals | Core IaC concepts, declarative vs imperative, tool landscape |
| [01-TERRAFORM.md](01-TERRAFORM.md) | Terraform | HCL fundamentals, providers, modules, state management |
| [03-CLOUDFORMATION.md](03-CLOUDFORMATION.md) | AWS CloudFormation | AWS-native IaC, nested stacks, drift detection |
| [04-STATE-MANAGEMENT.md](04-STATE-MANAGEMENT.md) | State Management | Remote state, locking, import, state surgery |
| [05-MODULES-AND-REUSE.md](05-MODULES-AND-REUSE.md) | Modules and Reuse | Module design, versioning, registries |
| [06-TESTING.md](06-TESTING.md) | IaC Testing | Policy as code, plan validation, integration testing |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Directory structure, naming, CI/CD integration |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial Pulumi documentation |
