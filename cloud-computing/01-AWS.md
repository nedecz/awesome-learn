# Amazon Web Services (AWS)

## Table of Contents

1. [Overview](#overview)
2. [AWS Global Infrastructure](#aws-global-infrastructure)
3. [Identity and Access Management (IAM)](#identity-and-access-management-iam)
4. [Compute Services](#compute-services)
5. [Storage Services](#storage-services)
6. [Networking](#networking)
7. [Database Services](#database-services)
8. [Serverless and Application Services](#serverless-and-application-services)
9. [Infrastructure as Code](#infrastructure-as-code)
10. [Monitoring and Logging](#monitoring-and-logging)
11. [Next Steps](#next-steps)
12. [Version History](#version-history)

---

## Overview

Amazon Web Services (AWS) is the world's most broadly adopted cloud platform, offering over 200 services from data centers globally. Launched in 2006 with S3 and EC2, AWS pioneered the public cloud model and continues to lead in both breadth of services and global infrastructure.

### Target Audience

- **Developers** building and deploying applications on AWS compute, storage, and database services
- **Architects** designing scalable, resilient cloud architectures using the Well-Architected Framework
- **DevOps Engineers** provisioning infrastructure with CloudFormation/CDK and managing CI/CD pipelines
- **Engineering Managers** evaluating AWS services, cost models, and cloud adoption strategies

### Scope

- AWS global infrastructure: regions, availability zones, edge locations
- IAM: users, groups, roles, policies, and best practices
- Compute: EC2, ECS, EKS, Lambda, and Auto Scaling
- Storage: S3, EBS, and EFS
- Networking: VPC, subnets, gateways, security groups, and NACLs
- Databases: RDS, Aurora, DynamoDB, and ElastiCache
- Serverless: API Gateway, Step Functions, SNS, SQS, EventBridge
- Infrastructure as code: CloudFormation and CDK
- Monitoring: CloudWatch, CloudTrail, and X-Ray

---

## AWS Global Infrastructure

AWS operates a global network of data centers organized into regions, availability zones, and edge locations. Understanding this hierarchy is essential for designing highly available, performant, and compliant applications.

### Regions

A **region** is a geographic area containing multiple isolated data center clusters. Each region is fully independent from other regions for maximum fault tolerance. As of 2025, AWS operates 30+ regions worldwide. Each is identified by a code such as `us-east-1` (N. Virginia), `eu-west-1` (Ireland), or `ap-southeast-1` (Singapore).

### Availability Zones

Each region has a minimum of three **Availability Zones (AZs)**. An AZ is one or more data centers with redundant power, networking, and connectivity. AZs within a region are connected via low-latency links but physically separated to protect against localized failures.

```
AWS Region: us-east-1 (N. Virginia)
─────────────────────────────────────────────
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ AZ: 1a      │  │ AZ: 1b      │  │ AZ: 1c      │
│ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │
│ │Data Ctrs│ │  │ │Data Ctrs│ │  │ │Data Ctrs│ │
│ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       └──── Low-Latency Links ──────────┘
```

Deploying across multiple AZs is a foundational best practice for high availability.

### Edge Locations

**Edge locations** are data centers used by CloudFront (CDN), Route 53 (DNS), and AWS WAF. With 400+ locations globally, they cache content closer to end users, reducing latency for static assets and API responses.

### Choosing a Region

Selecting the right AWS region involves balancing:

| Factor | Consideration |
|--------|--------------|
| **Latency** | Choose a region close to your users |
| **Compliance** | Data residency laws (e.g., GDPR) may require specific regions |
| **Service availability** | Not all services launch in every region simultaneously |
| **Cost** | Pricing varies by region — `us-east-1` is typically cheapest |

---

## Identity and Access Management (IAM)

AWS Identity and Access Management (IAM) is a global service that controls authentication (who can sign in) and authorization (what permissions they have) for every AWS resource. IAM is free to use.

### Users, Groups, and Roles

IAM provides three principal types:

- **Users** represent individual people or applications. Each user has unique credentials (password for console, access keys for CLI/API).
- **Groups** are collections of users. Permissions attached to a group are inherited by all members.
- **Roles** are identities assumed temporarily. Preferred for granting access to AWS services (e.g., EC2 reading S3), cross-account access, and federated users.

```
IAM Identity Hierarchy
──────────────────────
AWS Account (Root User)
 ├── User: alice ──► Group: Developers ──► Policy: DeveloperAccess
 ├── User: bob   ──► Group: Developers, Admins
 └── Role: EC2-S3-ReadRole ──► Policy: S3ReadOnlyAccess
       └── Trust: EC2 service can assume this role
```

### Policies and Permissions

Permissions in IAM are defined by **policies** — JSON documents specifying which actions are allowed or denied on which resources.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadAccess",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::my-application-bucket",
        "arn:aws:s3:::my-application-bucket/*"
      ]
    },
    {
      "Sid": "DenyDeleteOperations",
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::my-application-bucket/*"
    }
  ]
}
```

Each statement contains: **Effect** (`Allow`/`Deny`), **Action** (API operations like `s3:GetObject`), **Resource** (ARNs), and optional **Condition** (source IP, MFA, time).

### Principle of Least Privilege

The **principle of least privilege** means granting only the minimum permissions required for a task. Start with zero permissions and add incrementally. Use IAM Access Analyzer and the policy simulator to identify overly broad permissions.

### IAM Best Practices

- **Never use the root account** for day-to-day operations. Secure it with MFA.
- **Enable MFA** on all human user accounts, especially administrators.
- **Prefer roles over long-lived access keys.** Roles provide temporary, auto-rotating credentials.
- **Use groups** to assign permissions rather than attaching policies to individual users.
- **Rotate credentials** on a defined schedule and audit with Credential Reports.
- **Use AWS Organizations SCPs** for permission guardrails across multiple accounts.

---

## Compute Services

AWS offers compute services from full virtual machines to serverless functions.

### EC2 (Elastic Compute Cloud)

Amazon EC2 provides resizable virtual machines (instances) in the cloud with full OS, networking, and storage control.

**Instance types** define the hardware profile. They follow a naming convention: `m5.xlarge` = family `m` (general purpose), generation `5`, size `xlarge`.

| Family | Purpose | Example Use Cases |
|--------|---------|-------------------|
| `t3/t4g` | Burstable | Dev/test, small workloads |
| `m5/m6i` | General purpose | Web/app servers, databases |
| `c5/c6i` | Compute optimized | Batch processing, encoding |
| `r5/r6i` | Memory optimized | In-memory caches, analytics |
| `p4/p5` | GPU accelerated | ML training, HPC |
| `i3/i4i` | Storage optimized | OLTP, data warehousing |

- **AMIs (Amazon Machine Images)** are templates containing the OS and applications. Use AWS-provided AMIs, marketplace AMIs, or build custom AMIs with EC2 Image Builder.
- **Key pairs** provide SSH access to Linux instances. AWS stores the public key; you keep the private key.
- **Security groups** act as virtual firewalls controlling inbound and outbound traffic at the instance level.

### Container Services (ECS and EKS)

AWS provides two managed container orchestration services:
- **ECS (Elastic Container Service)** is AWS's container orchestration platform. Supports **EC2** launch type (you manage instances) and **Fargate** (serverless containers).
- **EKS (Elastic Kubernetes Service)** is a managed Kubernetes control plane. Choose EKS for Kubernetes expertise, multi-cloud portability, or ecosystem tools (Helm, Argo, Istio).

### Lambda (Serverless Compute)

AWS Lambda runs code in response to events without managing servers. You pay only for compute time consumed (billed per millisecond). Supports Node.js, Python, Java, Go, .NET, Ruby, and custom runtimes.

```bash
# Deploy a Lambda function using the AWS CLI
aws lambda create-function \
  --function-name my-function \
  --runtime python3.12 \
  --role arn:aws:iam::123456789012:role/lambda-role \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip
```

**Key limits:** 15-minute max execution, 128 MB–10 GB memory (CPU scales proportionally), 1,000 default concurrent executions per region. Event sources include API Gateway, S3, DynamoDB Streams, SQS, EventBridge, and many more.

### Auto Scaling

**EC2 Auto Scaling** adjusts the number of instances based on demand using CloudWatch metrics.

- **Launch template** — specifies AMI, instance type, key pair, security groups
- **Auto Scaling group (ASG)** — defines min, max, and desired instance count
- **Scaling policies** — target tracking, step scaling, or scheduled scaling

### Compute Service Comparison

| Criteria | EC2 | ECS/Fargate | Lambda |
|----------|-----|-------------|--------|
| **Abstraction** | Virtual machines | Containers | Functions |
| **Control** | Full OS access | Container-level | Code only |
| **Startup time** | Minutes | Seconds | Milliseconds |
| **Max runtime** | Unlimited | Unlimited | 15 minutes |
| **Pricing** | Per-second (instance) | Per-second (vCPU/mem) | Per-ms (invocation) |
| **Best for** | Legacy apps, full OS control | Microservices, batch | Event-driven, short tasks |

---

## Storage Services

AWS provides storage services for different access patterns — object, block, and file storage.

### S3 (Simple Storage Service)

Amazon S3 is an object storage service offering virtually unlimited storage with 99.999999999% (11 nines) durability. Data is organized into **buckets** (globally unique containers) and **objects** (files stored with a key and metadata).

**Key features:** versioning, server-side encryption (SSE-S3/KMS/C), lifecycle policies for automatic tier transitions, event notifications to Lambda/SQS/SNS.

```bash
# Create a bucket, upload a file, and enable versioning
aws s3 mb s3://my-app-data-bucket-2025 --region us-east-1
aws s3 cp ./report.pdf s3://my-app-data-bucket-2025/reports/
aws s3api put-bucket-versioning \
  --bucket my-app-data-bucket-2025 \
  --versioning-configuration Status=Enabled
```

### S3 Storage Classes

| Storage Class | Min Duration | Retrieval Fee | Use Case |
|---------------|-------------|---------------|----------|
| **S3 Standard** | None | None | Frequently accessed data, websites |
| **S3 Intelligent-Tiering** | None | None | Unknown or changing access patterns |
| **S3 Standard-IA** | 30 days | Per-GB | Infrequent access, rapid retrieval needed |
| **S3 One Zone-IA** | 30 days | Per-GB | Reproducible infrequent data |
| **S3 Glacier Instant** | 90 days | Per-GB | Archive with millisecond access |
| **S3 Glacier Flexible** | 90 days | Per-GB | Archives, minutes-to-hours retrieval |
| **S3 Glacier Deep Archive** | 180 days | Per-GB | Long-term retention (7–10 year) |

**Storage cost decreases from Standard to Deep Archive, but retrieval costs and latency increase.** Use lifecycle policies to migrate objects through tiers as they age.

### EBS (Elastic Block Store)

Amazon EBS provides persistent block storage volumes for EC2 instances, replicated within their AZ.

| Volume Type | Max IOPS | Max Throughput | Use Case |
|-------------|----------|----------------|----------|
| `gp3` (SSD) | 16,000 | 1,000 MB/s | Boot volumes, most workloads |
| `io2 Block Express` (SSD) | 256,000 | 4,000 MB/s | Large databases, latency-sensitive |
| `st1` (HDD) | 500 | 500 MB/s | Big data, log processing |
| `sc1` (HDD) | 250 | 250 MB/s | Infrequent access, lowest cost |

### EFS (Elastic File System)

Amazon EFS is a fully managed, elastic NFS file system shared across multiple EC2 instances and containers. EFS scales automatically and supports both Standard and Infrequent Access storage classes. Ideal for shared application data, content management, and ML training data.

---

## Networking

AWS networking is built around the Virtual Private Cloud (VPC), providing an isolated network environment.

### VPC (Virtual Private Cloud)

A **VPC** is a logically isolated virtual network. When you create a VPC, you specify an IPv4 CIDR block (e.g., `10.0.0.0/16`) that determines the available private IP addresses. Production workloads should use custom VPCs with intentional network design.

### Subnets

A **subnet** is a range of IP addresses within a VPC, associated with a single AZ:

- **Public subnets** have a route to an Internet Gateway. Use for load balancers, bastion hosts, and NAT gateways.
- **Private subnets** have no direct internet route. Resources access the internet through a NAT Gateway. Use for application servers, databases, and internal services.

### Internet Gateway and NAT Gateway

- **Internet Gateway (IGW)** — enables communication between VPC resources and the internet. One IGW per VPC.
- **NAT Gateway** — allows private subnet instances to initiate outbound internet traffic while blocking inbound connections. Deploy one per AZ for high availability.

### Security Groups vs NACLs

AWS provides two layers of network security:| Feature | Security Group | Network ACL (NACL) |
|---------|---------------|-------------------|
| **Level** | Instance (ENI) | Subnet |
| **State** | Stateful (return traffic auto-allowed) | Stateless (explicit in/out rules) |
| **Rules** | Allow only | Allow and deny |
| **Evaluation** | All rules together | Ordered (lowest number first) |
| **Default** | Deny inbound, allow outbound | Allow all |

### Route Tables

Every subnet has a **route table** directing traffic. Key routes: `10.0.0.0/16 → local` (VPC-internal), `0.0.0.0/0 → igw-xxx` (public subnets), `0.0.0.0/0 → nat-xxx` (private subnets).

### VPC Architecture Diagram

```
VPC: 10.0.0.0/16
┌──────────────────────────────────────────────────────┐
│                      ┌─────┐                         │
│                      │ IGW │◄────► Internet           │
│                      └──┬──┘                         │
│  ┌─ AZ: us-east-1a ────┼────────────────────────┐   │
│  │  Public: 10.0.1.0/24                          │   │
│  │  ┌───────────┐  ┌──────────────┐             │   │
│  │  │NAT Gateway│  │Load Balancer │             │   │
│  │  └─────┬─────┘  └──────────────┘             │   │
│  │  ──────┼──────────────────────────────────   │   │
│  │  Private: 10.0.2.0/24  (App Servers)         │   │
│  │  Private: 10.0.3.0/24  (RDS, ElastiCache)    │   │
│  └───────────────────────────────────────────────┘   │
│  ┌─ AZ: us-east-1b (mirrors AZ-a for HA) ───────┐   │
│  │  Public / Private subnets with standby DBs    │   │
│  └───────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

---

## Database Services

AWS offers purpose-built database services for different data models and access patterns.

### RDS (Relational Database Service)

Amazon RDS is a managed relational database service that handles provisioning, patching, backups, and failover. Supported engines: **Aurora** (MySQL/PostgreSQL compatible), **MySQL**, **PostgreSQL**, **MariaDB**, **Oracle**, and **SQL Server**.

**High availability:**

- **Multi-AZ deployments** — synchronous standby replica in a different AZ with automatic failover (60–120 seconds)
- **Read replicas** — asynchronous replicas that offload read traffic (up to 5 for most engines, 15 for Aurora)

### Amazon Aurora

Aurora is AWS's cloud-native relational database (MySQL/PostgreSQL compatible). It separates compute from storage and replicates data six ways across three AZs. Offers up to 5x MySQL throughput, auto-scaling storage (10 GB–128 TB), Serverless v2 for automatic compute scaling, and global databases with sub-second replication.

### DynamoDB

Amazon DynamoDB is a fully managed NoSQL database for single-digit millisecond performance at any scale. Key concepts:

- **Partition key** — attribute used to distribute data across partitions. Choose high-cardinality attributes (e.g., `userId`, `orderId`).
- **Sort key** (optional) — combined with partition key for composite primary keys and range queries.
- **Global Secondary Indexes (GSIs)** — enable querying on non-primary-key attributes with separate partition/sort keys.
- **Capacity modes** — on-demand (pay per request) or provisioned (with optional auto scaling).

### ElastiCache

Amazon ElastiCache provides fully managed in-memory data stores:

- **ElastiCache for Redis** — supports complex data structures, persistence, replication, clustering, and pub/sub. Use for session stores, leaderboards, and real-time analytics.
- **ElastiCache for Memcached** — simpler, multi-threaded cache for straightforward key-value caching. Use when persistence is not required.

### Database Service Comparison

| Criteria | RDS/Aurora | DynamoDB | ElastiCache |
|----------|-----------|----------|-------------|
| **Data model** | Relational (SQL) | Key-value / Document | Key-value (in-memory) |
| **Scaling** | Vertical + read replicas | Horizontal (auto) | Cluster sharding |
| **Latency** | Low ms | Single-digit ms | Sub-millisecond |
| **Best for** | Complex queries, joins, ACID | High-throughput key access | Caching, sessions, leaderboards |

---

## Serverless and Application Services

AWS offers a suite of serverless and integration services for building event-driven architectures.

### Lambda

AWS Lambda (covered in [Compute Services](#lambda-serverless-compute)) is the core serverless compute service, triggered by events from over 200 AWS services. Common patterns: processing S3 uploads, handling API Gateway requests, consuming SQS messages, and responding to DynamoDB Streams.

### API Gateway

Amazon API Gateway is a managed service for creating REST, HTTP, and WebSocket APIs.

- **REST APIs** — feature-rich with validation, transformation, caching, and usage plans
- **HTTP APIs** — lower cost and latency, ideal for Lambda or HTTP backend proxying
- **WebSocket APIs** — two-way communication for real-time applications

API Gateway handles throttling, authentication (IAM, Cognito, Lambda authorizers), and generates OpenAPI documentation.

### Step Functions

AWS Step Functions is a serverless orchestration service for coordinating multiple AWS services into visual workflows defined using the Amazon States Language (JSON). Use for multi-step pipelines, ETL orchestration, ML workflows, human approval processes, and distributed error handling.

### SNS (Simple Notification Service)

Amazon SNS is a fully managed **pub/sub** messaging service:

- **Topics** — logical access points for publishing messages
- **Subscriptions** — endpoints (Lambda, SQS, HTTP/S, email, SMS) that receive messages
- **Fan-out** — publish once, deliver to multiple subscribers simultaneously

### SQS (Simple Queue Service)

Amazon SQS is a fully managed **message queue** decoupling producers from consumers.

- **Standard queues** — nearly unlimited throughput, at-least-once delivery, best-effort ordering
- **FIFO queues** — exactly-once processing, strict ordering, up to 3,000 msg/s with batching
- **Dead-letter queues** — capture messages that fail after configured retry attempts

### EventBridge

Amazon EventBridge is a serverless event bus connecting applications using events from AWS services, SaaS partners, and custom sources. It supports event pattern matching, schema discovery, and event archive/replay. Use EventBridge for loose coupling with rich filtering and routing.

---

## Infrastructure as Code

Managing AWS resources through code ensures repeatability, version control, and auditability.

### CloudFormation

AWS CloudFormation lets you define infrastructure in YAML or JSON templates. It provisions resources in order, handles dependencies, and supports rollback on failure.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: S3 bucket with versioning and encryption

Resources:
  ApplicationBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-app-data-bucket-2025
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: aws:kms

Outputs:
  BucketArn:
    Value: !GetAtt ApplicationBucket.Arn
```

**Key concepts:**

- **Templates** — declarative YAML/JSON files describing resources
- **Stacks** — a single unit of provisioned resources from a template
- **Change sets** — preview changes before applying
- **Drift detection** — identify out-of-band modifications

### AWS CDK

The **AWS Cloud Development Kit (CDK)** lets you define infrastructure using programming languages (TypeScript, Python, Java, C#, Go). CDK synthesizes your code into CloudFormation templates.

```bash
# Initialize a CDK project and deploy
npx cdk init app --language typescript
npx cdk synth   # Synthesize CloudFormation template
npx cdk deploy  # Deploy the stack
```

CDK is ideal for teams that prefer imperative programming or need reusable constructs with loops, conditionals, and object-oriented patterns.

---

## Monitoring and Logging

Visibility into your AWS environment is critical for operations, security, and performance.

### CloudWatch

Amazon CloudWatch is the central monitoring service for AWS.

- **Metrics** — data points from AWS services plus custom application metrics
- **Alarms** — trigger actions when a metric crosses a threshold
- **Logs** — collect and query logs from EC2, Lambda, ECS via the CloudWatch Agent
- **Dashboards** — real-time visualizations of metrics and logs
- **Logs Insights** — interactive query language for log analysis

```bash
# CloudWatch alarm: notify when CPU > 80% for 10 minutes
aws cloudwatch put-metric-alarm \
  --alarm-name high-cpu --metric-name CPUUtilization \
  --namespace AWS/EC2 --statistic Average \
  --period 300 --threshold 80 --evaluation-periods 2 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:alerts
```

### CloudTrail

AWS CloudTrail records every API call made in your AWS account — who, when, from where, and what changed. Essential for security auditing, compliance, and incident investigation.

- **Management events** — control plane operations (creating EC2 instances, modifying IAM policies)
- **Data events** — data plane operations (S3 object reads/writes, Lambda invocations)
- **Insights events** — anomaly detection for unusual API activity

### X-Ray

AWS X-Ray provides distributed tracing for debugging requests across services. It captures **traces** (end-to-end request paths), **segments** (per-service work), and **subsegments** (granular operations like DB queries). The **service map** visualizes latency and errors between components.

---

## Next Steps

Continue your cloud computing learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Cloud Computing Fundamentals | Core cloud concepts, service models, deployment models, provider comparison |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial AWS documentation |
