# AWS CloudFormation

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [What is CloudFormation](#what-is-cloudformation)
   - [AWS-Native Infrastructure as Code](#aws-native-infrastructure-as-code)
   - [How CloudFormation Works](#how-cloudformation-works)
   - [JSON vs YAML Templates](#json-vs-yaml-templates)
   - [Deep AWS Integration](#deep-aws-integration)
3. [Template Anatomy](#template-anatomy)
   - [AWSTemplateFormatVersion](#awstemplateformatversion)
   - [Description](#description)
   - [Parameters](#parameters)
   - [Mappings](#mappings)
   - [Conditions](#conditions)
   - [Resources](#resources)
   - [Outputs](#outputs)
   - [Complete Template Example](#complete-template-example)
4. [Resources in Depth](#resources-in-depth)
   - [Resource Types and Syntax](#resource-types-and-syntax)
   - [DependsOn](#dependson)
   - [DeletionPolicy](#deletionpolicy)
   - [UpdatePolicy](#updatepolicy)
   - [UpdateReplacePolicy](#updatereplacepolicy)
   - [Resource Attributes Summary](#resource-attributes-summary)
5. [Parameters and Mappings](#parameters-and-mappings)
   - [Parameter Types](#parameter-types)
   - [Constraints and Validation](#constraints-and-validation)
   - [Default Values and AllowedValues](#default-values-and-allowedvalues)
   - [SSM Parameter References](#ssm-parameter-references)
   - [Mappings for Region and Environment Lookups](#mappings-for-region-and-environment-lookups)
6. [Intrinsic Functions](#intrinsic-functions)
   - [Ref](#ref)
   - [Fn::GetAtt](#fngetatt)
   - [Fn::Sub](#fnsub)
   - [Fn::Join](#fnjoin)
   - [Fn::Select](#fnselect)
   - [Fn::If](#fnif)
   - [Fn::ImportValue](#fnimportvalue)
   - [Other Useful Functions](#other-useful-functions)
7. [Nested Stacks and Cross-Stack References](#nested-stacks-and-cross-stack-references)
   - [Cross-Stack References with Exports](#cross-stack-references-with-exports)
   - [Nested Stacks](#nested-stacks)
   - [Stack Sets](#stack-sets)
   - [Modular Template Design](#modular-template-design)
8. [Change Sets](#change-sets)
   - [What Are Change Sets](#what-are-change-sets)
   - [Change Set Workflow](#change-set-workflow)
   - [Previewing Changes Before Execution](#previewing-changes-before-execution)
9. [Drift Detection](#drift-detection)
   - [Detecting Out-of-Band Changes](#detecting-out-of-band-changes)
   - [Drift Status Types](#drift-status-types)
   - [Remediation Strategies](#remediation-strategies)
10. [Stack Policies](#stack-policies)
    - [Protecting Critical Resources](#protecting-critical-resources)
    - [Temporary Overrides](#temporary-overrides)
11. [AWS CDK](#aws-cdk)
    - [What is the AWS CDK](#what-is-the-aws-cdk)
    - [Constructs: L1, L2, and L3](#constructs-l1-l2-and-l3)
    - [CDK TypeScript Examples](#cdk-typescript-examples)
    - [CDK Synth and Deploy](#cdk-synth-and-deploy)
12. [AWS SAM](#aws-sam)
    - [Serverless Application Model Overview](#serverless-application-model-overview)
    - [SAM Template Structure](#sam-template-structure)
    - [SAM CLI](#sam-cli)
13. [CloudFormation vs Terraform](#cloudformation-vs-terraform)
    - [Feature Comparison](#feature-comparison)
    - [When to Choose CloudFormation](#when-to-choose-cloudformation)
    - [When to Choose Terraform](#when-to-choose-terraform)
14. [CI/CD Integration](#cicd-integration)
    - [CodePipeline with CloudFormation](#codepipeline-with-cloudformation)
    - [GitHub Actions with CloudFormation](#github-actions-with-cloudformation)
15. [Next Steps](#next-steps)
16. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **AWS CloudFormation** — Amazon's native infrastructure as code service that enables teams to model, provision, and manage AWS resources using declarative JSON or YAML templates. It covers template anatomy, resource management, intrinsic functions, nested stacks, change sets, drift detection, stack policies, and higher-level abstractions like the AWS CDK and AWS SAM.

### Target Audience

- **Developers** provisioning AWS resources alongside their application code and building serverless applications
- **DevOps Engineers** designing and maintaining CloudFormation templates, nested stacks, and deployment pipelines
- **Platform Engineers** building reusable template libraries, stack sets, and self-service infrastructure patterns on AWS
- **Site Reliability Engineers (SREs)** managing stack drift, change sets, stack policies, and operational consistency across AWS accounts
- **Architects** evaluating CloudFormation strategies, multi-account patterns, and organizational standards for AWS infrastructure management

### Scope

- What CloudFormation is, how it works, and its deep integration with the AWS ecosystem
- Template anatomy: format version, description, parameters, mappings, conditions, resources, and outputs
- Resource management: types, dependencies, deletion policies, and update behaviors
- Parameters, mappings, and intrinsic functions for dynamic template authoring
- Nested stacks, cross-stack references, and stack sets for modular infrastructure
- Change sets for previewing updates and drift detection for identifying out-of-band changes
- Stack policies for protecting critical resources during updates
- AWS CDK for defining CloudFormation templates with TypeScript, Python, and other languages
- AWS SAM for serverless application deployment
- CloudFormation vs Terraform comparison and CI/CD integration patterns

---

## What is CloudFormation

### AWS-Native Infrastructure as Code

AWS CloudFormation is a **fully managed infrastructure as code service** provided by Amazon Web Services. Unlike third-party tools, CloudFormation is built directly into the AWS platform — it has first-class support for every AWS service, often on the same day a new service launches.

CloudFormation lets you define your entire AWS infrastructure in **declarative templates** written in JSON or YAML. You describe the desired end state of your resources, and CloudFormation handles the provisioning, configuration, and dependency resolution automatically.

Key characteristics of CloudFormation:

- **AWS-native** — no additional tooling, agents, or state files to manage
- **Declarative** — you describe what you want, not how to create it
- **Dependency resolution** — CloudFormation determines the correct creation order
- **Rollback support** — automatic rollback on failure to maintain consistency
- **No additional cost** — you pay only for the AWS resources created, not for CloudFormation itself

### How CloudFormation Works

```
┌───────────┐     ┌─────────────┐     ┌────────────────┐     ┌──────────────┐
│  Author    │     │  Upload to   │     │ CloudFormation  │     │  Provisioned  │
│  Template  │────▶│  S3/Console  │────▶│ Processes       │────▶│  AWS          │
│  (YAML)    │     │             │     │ Template        │     │  Resources    │
└───────────┘     └─────────────┘     └────────────────┘     └──────────────┘
```

The core concepts:

- **Template** — a JSON or YAML file that describes the AWS resources you want
- **Stack** — a running instance of a template; the collection of provisioned resources
- **Stack Event** — a log of each resource creation, update, or deletion
- **Change Set** — a preview of what changes will occur before executing an update

### JSON vs YAML Templates

CloudFormation supports both JSON and YAML. YAML is strongly preferred for its readability, support for comments, and concise syntax.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Simple S3 bucket

Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-example-bucket
```

| Feature | JSON | YAML |
|---|---|---|
| Comments | Not supported | Supported with `#` |
| Readability | Verbose, deeply nested | Clean, indentation-based |
| Intrinsic functions | `{"Fn::Sub": "..."}` | `!Sub "..."` shorthand |
| Multi-line strings | Escaped newlines | Block scalars (`\|`, `>`) |
| Adoption | Legacy templates | Industry standard |

> **Best Practice:** Use YAML for all new CloudFormation templates. The short-form intrinsic functions (`!Ref`, `!Sub`, `!GetAtt`) make templates significantly more readable.

### Deep AWS Integration

CloudFormation's primary advantage is its native position within the AWS ecosystem:

- **Same-day support** — new AWS services and features are available in CloudFormation on launch day
- **IAM integration** — uses your existing IAM roles and policies for stack operations
- **CloudTrail logging** — all stack operations are logged for auditing and compliance
- **AWS Organizations** — stack sets deploy across multiple accounts and regions
- **Service Catalog** — publish CloudFormation templates as self-service products for teams

---

## Template Anatomy

A CloudFormation template has seven top-level sections. Only `Resources` is required.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: String

Parameters:
  # Input values provided at stack creation/update time

Mappings:
  # Static key-value lookup tables

Conditions:
  # Conditional logic for resource creation

Resources:
  # AWS resources to provision (REQUIRED)

Outputs:
  # Values to return after stack creation
```

### AWSTemplateFormatVersion

The `AWSTemplateFormatVersion` identifies the template format capabilities. Currently, the only valid value is `"2010-09-09"`. While optional, including it is a best practice for clarity.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
```

### Description

A text string that describes the template. This appears in the AWS Console when viewing stacks.

```yaml
Description: >
  Production VPC with public and private subnets,
  NAT gateways, and VPN connectivity.
```

### Parameters

Parameters let you pass values into a template at stack creation or update time, making templates reusable across environments.

```yaml
Parameters:
  EnvironmentType:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, production]
    Description: The environment type for this stack

  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.small, t3.medium]

  VpcCidr:
    Type: String
    Default: "10.0.0.0/16"
    AllowedPattern: '(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})/(\d{1,2})'
```

### Mappings

Mappings define static lookup tables. They are commonly used to map regions to AMI IDs or environments to configuration values.

```yaml
Mappings:
  RegionAMI:
    us-east-1:
      HVM64: ami-0abcdef1234567890
    us-west-2:
      HVM64: ami-0fedcba0987654321

  EnvironmentConfig:
    dev:
      InstanceType: t3.micro
    production:
      InstanceType: t3.large
```

### Conditions

Conditions control whether certain resources are created or properties are assigned based on parameter values.

```yaml
Conditions:
  IsProduction: !Equals [!Ref EnvironmentType, production]
  CreateReadReplica: !And
    - !Equals [!Ref EnvironmentType, production]
    - !Equals [!Ref EnableReadReplica, "true"]
```

### Resources

The `Resources` section is the only required section. It defines the AWS resources to create.

```yaml
Resources:
  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP and HTTPS
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0

  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: !FindInMap [RegionAMI, !Ref "AWS::Region", HVM64]
      SecurityGroupIds:
        - !Ref WebServerSecurityGroup
```

### Outputs

Outputs declare values that can be viewed in the console, returned by `describe-stacks`, or exported for cross-stack references.

```yaml
Outputs:
  WebServerPublicIP:
    Description: Public IP address of the web server
    Value: !GetAtt WebServer.PublicIp

  VPCId:
    Value: !Ref VPC
    Export:
      Name: !Sub "${AWS::StackName}-VPCId"
```

### Complete Template Example

Here is a complete template combining all sections:

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Web application stack with VPC, EC2, and RDS

Parameters:
  EnvironmentType:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, production]
  DBPassword:
    Type: String
    NoEcho: true
    MinLength: 8

Mappings:
  EnvironmentConfig:
    dev:
      InstanceType: t3.micro
    production:
      InstanceType: t3.large

Conditions:
  IsProduction: !Equals [!Ref EnvironmentType, production]

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: "10.0.0.0/16"
      Tags:
        - Key: Name
          Value: !Sub "${EnvironmentType}-vpc"

  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !FindInMap [EnvironmentConfig, !Ref EnvironmentType, InstanceType]
      ImageId: ami-0abcdef1234567890

  Database:
    Type: AWS::RDS::DBInstance
    Condition: IsProduction
    DeletionPolicy: Snapshot
    Properties:
      Engine: mysql
      MasterUsername: admin
      MasterUserPassword: !Ref DBPassword

Outputs:
  VPCId:
    Value: !Ref VPC
    Export:
      Name: !Sub "${AWS::StackName}-VPCId"
```

---

## Resources in Depth

### Resource Types and Syntax

Every CloudFormation resource follows the pattern `AWS::Service::Resource`. There are over 900 resource types covering the full range of AWS services.

```yaml
Resources:
  LogicalResourceName:
    Type: AWS::Service::Resource
    Properties:
      PropertyName: PropertyValue
```

Common resource types:

| Resource Type | Service | Description |
|---|---|---|
| `AWS::EC2::Instance` | EC2 | Virtual machine instance |
| `AWS::EC2::VPC` | VPC | Virtual private cloud |
| `AWS::S3::Bucket` | S3 | Object storage bucket |
| `AWS::RDS::DBInstance` | RDS | Relational database instance |
| `AWS::Lambda::Function` | Lambda | Serverless function |
| `AWS::DynamoDB::Table` | DynamoDB | NoSQL table |
| `AWS::IAM::Role` | IAM | Identity and access role |

### DependsOn

CloudFormation automatically determines resource creation order from `!Ref` and `!GetAtt` references. Use `DependsOn` when there is an implicit dependency that CloudFormation cannot detect.

```yaml
Resources:
  ECSService:
    Type: AWS::ECS::Service
    DependsOn: ALBListener
    Properties:
      Cluster: !Ref ECSCluster
      DesiredCount: 2
```

> **Best Practice:** Avoid unnecessary `DependsOn` declarations. CloudFormation resolves most dependencies automatically from references. Use `DependsOn` only for implicit ordering that cannot be expressed through resource references.

### DeletionPolicy

`DeletionPolicy` controls what happens to a resource when it is removed from a template or when the stack is deleted.

```yaml
Resources:
  ProductionDatabase:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot
    Properties:
      DBInstanceClass: db.r5.large
      Engine: postgres

  CriticalBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
```

| DeletionPolicy | Behavior | Use Case |
|---|---|---|
| `Delete` | Resource is deleted (default) | Temporary and non-critical resources |
| `Retain` | Resource is preserved after stack deletion | Production databases, S3 buckets with data |
| `Snapshot` | Creates a snapshot before deletion | RDS instances, EBS volumes, Redshift clusters |

### UpdatePolicy

`UpdatePolicy` controls how CloudFormation handles updates to Auto Scaling groups, Lambda aliases, and ElastiCache replication groups.

```yaml
Resources:
  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    UpdatePolicy:
      AutoScalingRollingUpdate:
        MaxBatchSize: 2
        MinInstancesInService: 2
        PauseTime: PT5M
        WaitOnResourceSignals: true
    Properties:
      MinSize: 2
      MaxSize: 10
```

### UpdateReplacePolicy

`UpdateReplacePolicy` controls what happens to the existing physical resource when CloudFormation replaces it during a stack update.

```yaml
Resources:
  MyDatabase:
    Type: AWS::RDS::DBInstance
    UpdateReplacePolicy: Snapshot
    DeletionPolicy: Snapshot
    Properties:
      DBInstanceClass: db.r5.large
      Engine: mysql
```

| UpdateReplacePolicy | Behavior |
|---|---|
| `Delete` | Old resource is deleted after replacement (default) |
| `Retain` | Old resource is preserved after replacement |
| `Snapshot` | Old resource snapshot is created before deletion |

### Resource Attributes Summary

| Attribute | Purpose |
|---|---|
| `DependsOn` | Explicit ordering between resources |
| `Condition` | Conditionally create the resource |
| `DeletionPolicy` | Control behavior on stack deletion |
| `UpdatePolicy` | Control behavior during updates |
| `UpdateReplacePolicy` | Control behavior when resource is replaced |

---

## Parameters and Mappings

### Parameter Types

CloudFormation supports several parameter types for input validation:

| Type | Description | Example |
|---|---|---|
| `String` | A literal string | `"my-bucket"` |
| `Number` | An integer or float | `42` |
| `CommaDelimitedList` | Comma-separated strings | `"subnet-a,subnet-b"` |
| `AWS::EC2::VPC::Id` | Existing VPC ID | `vpc-0abc123` |
| `AWS::EC2::Subnet::Id` | Existing subnet ID | `subnet-0abc123` |
| `AWS::EC2::KeyPair::KeyName` | Existing key pair name | `my-key-pair` |
| `AWS::SSM::Parameter::Value<String>` | SSM Parameter Store value | See SSM section |
| `List<AWS::EC2::Subnet::Id>` | List of existing subnet IDs | Multiple subnets |

### Constraints and Validation

Parameters support multiple validation constraints to catch errors at stack creation time rather than during resource provisioning.

```yaml
Parameters:
  DatabaseName:
    Type: String
    MinLength: 3
    MaxLength: 63
    AllowedPattern: "[a-zA-Z][a-zA-Z0-9]*"
    ConstraintDescription: Must begin with a letter and contain only alphanumeric characters.

  InstanceCount:
    Type: Number
    MinValue: 1
    MaxValue: 20
    Default: 2
```

### Default Values and AllowedValues

Use `Default` for sensible defaults and `AllowedValues` for strict enumeration:

```yaml
Parameters:
  Environment:
    Type: String
    Default: dev
    AllowedValues: [dev, staging, production]

  DatabaseEngine:
    Type: String
    Default: mysql
    AllowedValues: [mysql, postgres, aurora-mysql, aurora-postgresql]

  EnableMonitoring:
    Type: String
    Default: "true"
    AllowedValues: ["true", "false"]
```

### SSM Parameter References

CloudFormation can resolve values from AWS Systems Manager Parameter Store at stack creation time. This eliminates hardcoding values like AMI IDs.

```yaml
Parameters:
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2

Resources:
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: t3.micro
```

> **Best Practice:** Use SSM parameter types for AMI IDs. This ensures templates always reference the latest approved AMI without manual updates.

### Mappings for Region and Environment Lookups

Mappings are ideal for values that vary by region, environment, or account but are known in advance.

```yaml
Mappings:
  RegionConfig:
    us-east-1:
      AMI: ami-0abcdef1234567890
    us-west-2:
      AMI: ami-0fedcba0987654321
    eu-west-1:
      AMI: ami-0aabbccdd11223344

  EnvironmentSize:
    small:
      EC2: t3.micro
      RDS: db.t3.micro
    large:
      EC2: t3.large
      RDS: db.r5.large

Resources:
  WebServer:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap [RegionConfig, !Ref "AWS::Region", AMI]
      InstanceType: !FindInMap [EnvironmentSize, !Ref EnvironmentSize, EC2]
```

---

## Intrinsic Functions

Intrinsic functions are built-in functions that CloudFormation provides for dynamic value resolution within templates. YAML supports short-form syntax with `!` prefix.

### Ref

`Ref` returns the value of a parameter or the physical ID of a resource.

```yaml
# Reference a parameter
InstanceType: !Ref InstanceType

# Reference a resource (returns its physical ID)
VpcId: !Ref MyVPC

# Pseudo-parameters
Region: !Ref "AWS::Region"
AccountId: !Ref "AWS::AccountId"
StackName: !Ref "AWS::StackName"
```

Common pseudo-parameters:

| Pseudo-Parameter | Returns |
|---|---|
| `AWS::AccountId` | AWS account ID (e.g., `123456789012`) |
| `AWS::Region` | Current region (e.g., `us-east-1`) |
| `AWS::StackName` | Stack name |
| `AWS::StackId` | Stack ARN |
| `AWS::URLSuffix` | Domain suffix (e.g., `amazonaws.com`) |
| `AWS::NoValue` | Removes a property when used with conditions |

### Fn::GetAtt

`Fn::GetAtt` returns an attribute of a resource. Unlike `Ref`, which returns the primary identifier, `GetAtt` can access any exposed attribute.

```yaml
Outputs:
  BucketArn:
    Value: !GetAtt MyBucket.Arn
  InstancePrivateIp:
    Value: !GetAtt WebServer.PrivateIp
  DBEndpoint:
    Value: !GetAtt Database.Endpoint.Address
```

### Fn::Sub

`Fn::Sub` substitutes variables in a string. It supports both template parameter references and resource references.

```yaml
# Simple substitution with resource and parameter references
Tags:
  - Key: Name
    Value: !Sub "${EnvironmentType}-web-${AWS::Region}"

# Multi-line string with substitution
UserData:
  Fn::Base64: !Sub |
    #!/bin/bash -xe
    aws s3 cp s3://${ConfigBucket}/config.yml /etc/app/
    echo "ENVIRONMENT=${EnvironmentType}" >> /etc/environment
    systemctl start myapp

# Custom variable mapping
Body:
  Fn::Sub:
    - |
      openapi: "3.0"
      paths:
        /items:
          get:
            x-amazon-apigateway-integration:
              uri: ${LambdaArn}
    - LambdaArn: !GetAtt ItemsFunction.Arn
```

### Fn::Join

`Fn::Join` concatenates a list of values with a specified delimiter.

```yaml
Resource: !Join ["", ["arn:aws:s3:::", !Ref MyBucket, "/*"]]
```

> **Best Practice:** Prefer `!Sub` over `!Join` for string construction. `!Sub "arn:aws:s3:::${MyBucket}/*"` is cleaner than the equivalent `!Join` expression.

### Fn::Select

`Fn::Select` returns a single object from a list by index.

```yaml
AvailabilityZone: !Select [0, !GetAZs ""]
```

### Fn::If

`Fn::If` returns one of two values based on a condition.

```yaml
Conditions:
  IsProduction: !Equals [!Ref EnvironmentType, production]

Resources:
  Database:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceClass: !If [IsProduction, db.r5.large, db.t3.micro]
      MultiAZ: !If [IsProduction, true, false]
      # Remove Iops property entirely when not production
      Iops: !If [IsProduction, 3000, !Ref "AWS::NoValue"]
```

### Fn::ImportValue

`Fn::ImportValue` imports a value exported by another stack. This enables cross-stack references.

```yaml
# Stack A exports
Outputs:
  VPCId:
    Value: !Ref VPC
    Export:
      Name: shared-vpc-id

# Stack B imports
Resources:
  SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !ImportValue shared-vpc-id
```

### Other Useful Functions

```yaml
# Fn::Split — split a string into a list
Subnets: !Split [",", !Ref SubnetIdsParam]

# Fn::GetAZs — get availability zones for a region
AZ: !Select [0, !GetAZs ""]

# Fn::Cidr — generate CIDR address blocks
SubnetCidrs: !Cidr [!GetAtt VPC.CidrBlock, 6, 8]

# Fn::Base64 — encode a string to Base64
UserData: !Base64 |
  #!/bin/bash
  echo "Hello World"
```

---

## Nested Stacks and Cross-Stack References

### Cross-Stack References with Exports

Cross-stack references allow one stack to export values that other stacks consume using `Fn::ImportValue`.

```
┌──────────────────┐    Export     ┌──────────────────┐    Import    ┌──────────────────┐
│  Network Stack    │────────────▶│ Application Stack │◀────────────│  Database Stack   │
│  VPC, Subnets     │  VPCId      │ EC2, ALB          │  VPCId      │  RDS, ElastiCache │
└──────────────────┘  SubnetIds   └──────────────────┘  SubnetIds   └──────────────────┘
```

**Network stack (exports):**

```yaml
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: "10.0.0.0/16"

Outputs:
  VPCId:
    Value: !Ref VPC
    Export:
      Name: !Sub "${AWS::StackName}-VPCId"
  PrivateSubnetIds:
    Value: !Join [",", [!Ref PrivateSubnet1, !Ref PrivateSubnet2]]
    Export:
      Name: !Sub "${AWS::StackName}-PrivateSubnetIds"
```

**Application stack (imports):**

```yaml
Resources:
  AppSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !ImportValue
        Fn::Sub: "${NetworkStack}-VPCId"
      GroupDescription: Application security group
```

### Nested Stacks

Nested stacks allow you to compose a parent stack from child stacks using `AWS::CloudFormation::Stack`. Each child stack is a separate template stored in S3.

```yaml
Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-templates/network.yaml
      Parameters:
        EnvironmentType: !Ref EnvironmentType

  ComputeStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: NetworkStack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-templates/compute.yaml
      Parameters:
        VpcId: !GetAtt NetworkStack.Outputs.VPCId
        SubnetIds: !GetAtt NetworkStack.Outputs.PrivateSubnetIds

  DatabaseStack:
    Type: AWS::CloudFormation::Stack
    DependsOn: NetworkStack
    Properties:
      TemplateURL: https://s3.amazonaws.com/my-templates/database.yaml
      Parameters:
        VpcId: !GetAtt NetworkStack.Outputs.VPCId
```

### Stack Sets

Stack sets let you deploy a single CloudFormation template across multiple AWS accounts and regions from a central administrator account.

```bash
# Create a stack set
aws cloudformation create-stack-set \
  --stack-set-name security-baseline \
  --template-body file://security-baseline.yaml \
  --permission-model SERVICE_MANAGED

# Deploy to specific OUs and regions
aws cloudformation create-stack-instances \
  --stack-set-name security-baseline \
  --deployment-targets OrganizationalUnitIds='["ou-abc123"]' \
  --regions us-east-1 us-west-2 eu-west-1
```

Common use cases: security baselines across all accounts, networking (VPCs, Transit Gateway), centralized logging, and compliance enforcement.

### Modular Template Design

Design templates as modular, composable units. Each template should represent a single concern.

```
project/
├── templates/
│   ├── main.yaml              # Parent stack (orchestration)
│   ├── network/
│   │   └── vpc.yaml           # VPC, subnets, route tables
│   ├── compute/
│   │   └── ecs-cluster.yaml   # ECS cluster and services
│   ├── database/
│   │   └── rds.yaml           # RDS instances
│   └── monitoring/
│       └── alarms.yaml        # CloudWatch alarms
├── parameters/
│   ├── dev.json
│   ├── staging.json
│   └── production.json
└── scripts/
    └── deploy.sh
```

---

## Change Sets

### What Are Change Sets

A change set is a summary of proposed changes to a stack. It lets you preview exactly what CloudFormation will create, modify, or delete before executing the update. This is critical for production environments where unexpected changes could cause downtime.

### Change Set Workflow

```
┌──────────────┐     ┌───────────────┐     ┌──────────────────┐
│   Update      │     │  Create        │     │   Review Change   │
│   Template    │────▶│  Change Set    │────▶│   Set Summary     │
│               │     │               │     │                   │
└──────────────┘     └───────────────┘     └────────┬─────────┘
                                                     │
                                           ┌─────────┴─────────┐
                                           │                    │
                                           ▼                    ▼
                                    ┌─────────────┐    ┌──────────────┐
                                    │  Execute     │    │   Delete      │
                                    │  Change Set  │    │   Change Set  │
                                    │  (apply)     │    │   (cancel)    │
                                    └─────────────┘    └──────────────┘
```

### Previewing Changes Before Execution

```bash
# Create a change set
aws cloudformation create-change-set \
  --stack-name production-app \
  --template-body file://updated-template.yaml \
  --change-set-name update-instance-types

# Review the proposed changes
aws cloudformation describe-change-set \
  --stack-name production-app \
  --change-set-name update-instance-types

# Execute if changes look correct
aws cloudformation execute-change-set \
  --stack-name production-app \
  --change-set-name update-instance-types
```

The change set output shows the action for each resource:

| Action | Meaning |
|---|---|
| `Add` | A new resource will be created |
| `Modify` | An existing resource will be updated (may require replacement) |
| `Remove` | An existing resource will be deleted |
| `Import` | An existing AWS resource will be imported into the stack |

> **Best Practice:** Always use change sets for production stack updates. Never run `update-stack` directly in production — always create, review, and then execute a change set.

---

## Drift Detection

### Detecting Out-of-Band Changes

Drift occurs when the actual state of a resource differs from its expected state as defined in the CloudFormation template. This happens when someone modifies a resource directly through the AWS Console, CLI, or another tool instead of updating the CloudFormation stack.

```bash
# Initiate drift detection
aws cloudformation detect-stack-drift \
  --stack-name production-app

# Check detection status
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id <detection-id>

# View drift results
aws cloudformation describe-stack-resource-drifts \
  --stack-name production-app \
  --stack-resource-drift-status-filters MODIFIED DELETED
```

### Drift Status Types

| Status | Meaning |
|---|---|
| `IN_SYNC` | Resource matches the template definition |
| `MODIFIED` | One or more properties differ from the template |
| `DELETED` | Resource was deleted outside of CloudFormation |
| `NOT_CHECKED` | Drift detection has not been run on this resource |

At the stack level:

| Stack Drift Status | Meaning |
|---|---|
| `IN_SYNC` | All resources match their expected configuration |
| `DRIFTED` | One or more resources have drifted |
| `NOT_CHECKED` | Drift detection has not been performed |

### Remediation Strategies

When drift is detected, you have several options: update the template to match the current state (accept the drift), re-deploy the original template to revert changes, import resources recreated outside CloudFormation, or document and ignore intentional drift.

```bash
# Re-deploy the original template to revert drift
aws cloudformation update-stack \
  --stack-name production-app \
  --template-body file://original-template.yaml \
  --parameters ParameterKey=EnvironmentType,UsePreviousValue=true
```

> **Best Practice:** Run drift detection regularly — ideally as part of your CI/CD pipeline or on a scheduled basis with AWS Config rules. Catch drift early before it causes deployment failures.

---

## Stack Policies

### Protecting Critical Resources

A stack policy is a JSON document that defines which update actions are allowed on specific resources. Once applied, the stack policy prevents accidental updates or deletions of critical resources.

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "Update:*",
      "Principal": "*",
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": ["Update:Replace", "Update:Delete"],
      "Principal": "*",
      "Resource": "LogicalResourceId/ProductionDatabase"
    }
  ]
}
```

```bash
# Apply a stack policy
aws cloudformation set-stack-policy \
  --stack-name production-app \
  --stack-policy-body file://stack-policy.json
```

### Temporary Overrides

When you need to update a protected resource, you can temporarily override the stack policy during the update.

```bash
aws cloudformation update-stack \
  --stack-name production-app \
  --template-body file://updated-template.yaml \
  --stack-policy-during-update-body '{"Statement":[{"Effect":"Allow","Action":"Update:*","Principal":"*","Resource":"*"}]}'
```

> **Best Practice:** Apply stack policies to every production stack. Protect databases, stateful resources, and networking components from replacement or deletion. Use temporary overrides only after peer review of the proposed change.

---

## AWS CDK

### What is the AWS CDK

The **AWS Cloud Development Kit (CDK)** is an open-source framework that lets you define CloudFormation templates using general-purpose programming languages — TypeScript, Python, Java, C#, and Go. CDK code is **synthesized** into standard CloudFormation templates before deployment.

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│  CDK App      │     │  cdk synth    │     │  CloudFormation   │
│  (TypeScript, │────▶│  (generates   │────▶│  Template (YAML)  │
│   Python,     │     │   template)   │     │                   │
│   Java, etc.) │     │              │     │                   │
└──────────────┘     └──────────────┘     └────────┬─────────┘
                                                    │
                                                    ▼
                                          ┌──────────────────┐
                                          │  CloudFormation    │
                                          │  Stack (deployed   │
                                          │  AWS resources)    │
                                          └──────────────────┘
```

### Constructs: L1, L2, and L3

CDK organizes resources into three levels of abstraction called **constructs**:

| Level | Name | Description | Example |
|---|---|---|---|
| L1 | CFN Resources | Direct 1:1 mapping to CloudFormation resources | `CfnBucket` |
| L2 | Curated | Higher-level with sensible defaults and helper methods | `Bucket` |
| L3 | Patterns | Multi-resource architectures assembled as a single construct | `LambdaRestApi` |

- **L1 (CFN Resources)** — auto-generated from the CloudFormation spec, prefixed with `Cfn`. No opinions or defaults. Use when L2 is unavailable or you need full control.
- **L2 (Curated Constructs)** — handcrafted by the CDK team with best-practice defaults, convenience methods, and strong typing. Most common level for daily use.
- **L3 (Patterns)** — high-level constructs that wire together multiple L2 resources into common architectures (e.g., a load-balanced Fargate service).

### CDK TypeScript Examples

**Basic S3 bucket with versioning and encryption:**

```typescript
import * as cdk from "aws-cdk-lib";
import * as s3 from "aws-cdk-lib/aws-s3";
import { Construct } from "constructs";

class StorageStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const bucket = new s3.Bucket(this, "DataBucket", {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
    });

    new cdk.CfnOutput(this, "BucketArn", {
      value: bucket.bucketArn,
    });
  }
}
```

**VPC with public and private subnets:**

```typescript
import * as cdk from "aws-cdk-lib";
import * as ec2 from "aws-cdk-lib/aws-ec2";
import { Construct } from "constructs";

class NetworkStack extends cdk.Stack {
  public readonly vpc: ec2.Vpc;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    this.vpc = new ec2.Vpc(this, "AppVpc", {
      maxAzs: 3,
      natGateways: 1,
      subnetConfiguration: [
        { cidrMask: 24, name: "Public", subnetType: ec2.SubnetType.PUBLIC },
        { cidrMask: 24, name: "Private", subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      ],
    });
  }
}
```

**Lambda function with API Gateway (L3 pattern):**

```typescript
import * as cdk from "aws-cdk-lib";
import * as lambda from "aws-cdk-lib/aws-lambda";
import * as apigw from "aws-cdk-lib/aws-apigateway";
import { Construct } from "constructs";

class ApiStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const handler = new lambda.Function(this, "ApiHandler", {
      runtime: lambda.Runtime.NODEJS_18_X,
      code: lambda.Code.fromAsset("lambda"),
      handler: "index.handler",
      environment: { TABLE_NAME: "items" },
      timeout: cdk.Duration.seconds(30),
    });

    // L3 pattern: wires up API Gateway + Lambda integration
    const api = new apigw.LambdaRestApi(this, "ItemsApi", {
      handler: handler,
      proxy: false,
    });

    const items = api.root.addResource("items");
    items.addMethod("GET");
    items.addMethod("POST");
  }
}
```

### CDK Synth and Deploy

```bash
cdk init app --language typescript   # Initialize a new CDK project
cdk synth                            # Synthesize CloudFormation template
cdk diff                             # View diff between deployed and local
cdk deploy                           # Deploy the stack
cdk deploy --all                     # Deploy all stacks
cdk destroy                          # Destroy a stack
```

---

## AWS SAM

### Serverless Application Model Overview

The **AWS Serverless Application Model (SAM)** is an open-source framework for building serverless applications on AWS. SAM is an extension of CloudFormation — every SAM template is a valid CloudFormation template with additional shorthand resource types for serverless patterns.

SAM provides:

- **Shorthand syntax** for Lambda functions, API Gateway, DynamoDB tables, and other serverless resources
- **Local development** with `sam local` for invoking functions and running APIs locally
- **Built-in best practices** for serverless deployments (gradual rollouts, canary deployments)
- **SAM CLI** for building, packaging, deploying, and testing serverless applications

### SAM Template Structure

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: Serverless API with Lambda and DynamoDB

Globals:
  Function:
    Timeout: 30
    Runtime: nodejs18.x
    Environment:
      Variables:
        TABLE_NAME: !Ref ItemsTable

Resources:
  GetItemFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/handlers/getItem.handler
      CodeUri: .
      Events:
        GetItem:
          Type: Api
          Properties:
            Path: /items/{id}
            Method: get
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref ItemsTable

  PutItemFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/handlers/putItem.handler
      CodeUri: .
      Events:
        PutItem:
          Type: Api
          Properties:
            Path: /items
            Method: post
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref ItemsTable

  ItemsTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      PrimaryKey:
        Name: id
        Type: String

Outputs:
  ApiUrl:
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/"
```

SAM resource types vs standard CloudFormation:

| SAM Resource | Equivalent CloudFormation |
|---|---|
| `AWS::Serverless::Function` | `AWS::Lambda::Function` + IAM Role + Permissions |
| `AWS::Serverless::Api` | `AWS::ApiGateway::RestApi` + Stage + Deployment |
| `AWS::Serverless::SimpleTable` | `AWS::DynamoDB::Table` |
| `AWS::Serverless::HttpApi` | `AWS::ApiGatewayV2::Api` |

### SAM CLI

```bash
# Initialize a new SAM project
sam init --runtime nodejs18.x --app-template hello-world

# Build and invoke locally
sam build
sam local invoke GetItemFunction --event events/get-item.json
sam local start-api

# Validate and deploy
sam validate
sam deploy --guided

# Stream function logs
sam logs -n GetItemFunction --stack-name my-app --tail
```

---

## CloudFormation vs Terraform

### Feature Comparison

| Feature | CloudFormation | Terraform |
|---|---|---|
| **Provider** | AWS only | Multi-cloud (3000+ providers) |
| **Language** | JSON/YAML | HCL |
| **State management** | Managed by AWS | Self-managed state files |
| **Cost** | Free | Free (open source) / Paid (Cloud) |
| **New AWS features** | Same-day support | Days to weeks after launch |
| **Drift detection** | Built-in | `terraform plan` |
| **Rollback** | Automatic on failure | Manual |
| **Modularity** | Nested stacks, exports | Modules, workspaces |
| **Higher abstraction** | AWS CDK, AWS SAM | CDK for Terraform (CDKTF) |
| **Preview changes** | Change sets | `terraform plan` |
| **Multi-account** | Stack sets | Workspaces, provider aliases |

### When to Choose CloudFormation

- **AWS-only infrastructure** — no multi-cloud requirements
- **Same-day AWS feature support** is critical for your team
- **No state file management** — AWS manages all state automatically
- **Automatic rollback** on failure is a hard requirement
- **AWS Organizations integration** — stack sets for multi-account governance
- **Existing investment** in CloudFormation templates, CDK, or SAM
- **Compliance requirements** favor AWS-native tools with CloudTrail integration

### When to Choose Terraform

- **Multi-cloud or hybrid** — infrastructure spans AWS, Azure, GCP, or on-premises
- **Broader provider ecosystem** — managing SaaS services (Datadog, PagerDuty, GitHub)
- **Team already knows HCL** and has existing Terraform codebases
- **Stronger IDE experience** — HCL has better language tooling than YAML
- **`terraform plan`** provides richer preview output than change sets
- **Community modules** — large ecosystem of pre-built, battle-tested modules
- **State flexibility** — choose between S3, Terraform Cloud, or other backends

---

## CI/CD Integration

### CodePipeline with CloudFormation

AWS CodePipeline has native CloudFormation integration for deploying infrastructure changes through a managed pipeline.

```yaml
Resources:
  Pipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      Name: infrastructure-pipeline
      RoleArn: !GetAtt PipelineRole.Arn
      ArtifactStore:
        Type: S3
        Location: !Ref ArtifactBucket
      Stages:
        - Name: Source
          Actions:
            - Name: SourceAction
              ActionTypeId:
                Category: Source
                Owner: AWS
                Provider: CodeStarSourceConnection
                Version: "1"
              Configuration:
                ConnectionArn: !Ref CodeStarConnection
                FullRepositoryId: my-org/infrastructure
                BranchName: main
              OutputArtifacts:
                - Name: SourceOutput

        - Name: Staging
          Actions:
            - Name: CreateChangeSet
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CloudFormation
                Version: "1"
              Configuration:
                ActionMode: CHANGE_SET_REPLACE
                StackName: staging-app
                ChangeSetName: staging-changeset
                TemplatePath: SourceOutput::template.yaml
                RoleArn: !GetAtt CloudFormationRole.Arn
              InputArtifacts:
                - Name: SourceOutput
            - Name: ExecuteChangeSet
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CloudFormation
                Version: "1"
              Configuration:
                ActionMode: CHANGE_SET_EXECUTE
                StackName: staging-app
                ChangeSetName: staging-changeset

        - Name: Production
          Actions:
            - Name: DeployProduction
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CloudFormation
                Version: "1"
              Configuration:
                ActionMode: CREATE_UPDATE
                StackName: production-app
                TemplatePath: SourceOutput::template.yaml
                RoleArn: !GetAtt CloudFormationRole.Arn
              InputArtifacts:
                - Name: SourceOutput
```

### GitHub Actions with CloudFormation

```yaml
name: Deploy CloudFormation

on:
  push:
    branches: [main]
    paths: ["infrastructure/**"]

permissions:
  id-token: write
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1

      - name: Validate template
        run: aws cloudformation validate-template --template-body file://infrastructure/template.yaml

      - name: Lint with cfn-lint
        uses: scottbrenner/cfn-lint-action@v2
        with:
          command: cfn-lint infrastructure/**/*.yaml

  deploy:
    needs: validate
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1

      - name: Deploy stack
        uses: aws-actions/aws-cloudformation-github-deploy@v1
        with:
          name: production-app
          template: infrastructure/template.yaml
          parameter-overrides: "EnvironmentType=production"
          no-fail-on-empty-changeset: "1"
          capabilities: "CAPABILITY_IAM,CAPABILITY_NAMED_IAM"
```

---

## Next Steps

Continue your Infrastructure as Code learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | IaC Fundamentals | Core IaC concepts, declarative vs imperative, tool landscape |
| [01-TERRAFORM.md](01-TERRAFORM.md) | Terraform | HCL fundamentals, providers, modules, state management |
| [02-PULUMI.md](02-PULUMI.md) | Pulumi | General-purpose languages for IaC, stacks, automation API |
| [04-STATE-MANAGEMENT.md](04-STATE-MANAGEMENT.md) | State Management | Remote state, locking, import, state surgery |
| [05-MODULES-AND-REUSE.md](05-MODULES-AND-REUSE.md) | Modules and Reuse | Module design, versioning, registries |
| [06-TESTING.md](06-TESTING.md) | IaC Testing | Policy as code, plan validation, integration testing |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Directory structure, naming, CI/CD integration |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial CloudFormation documentation |