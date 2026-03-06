# Cloud Computing Learning Resources

A comprehensive guide to cloud computing — IaaS, PaaS, SaaS, core AWS/Azure/GCP services, networking, serverless, cost optimization, multi-cloud strategies, governance, and production best practices.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | Cloud computing models (IaaS, PaaS, SaaS), shared responsibility | **Start here** |
| [01-AWS](01-AWS.md) | Core AWS services, IAM, VPC, EC2, S3, Lambda | When building on Amazon Web Services |
| [02-AZURE](02-AZURE.md) | Core Azure services, Entra ID, App Service, Functions | When building on Microsoft Azure |
| [03-GCP](03-GCP.md) | Core GCP services, IAM, Cloud Run, Cloud Functions | When building on Google Cloud Platform |
| [04-NETWORKING](04-NETWORKING.md) | VPCs, subnets, peering, DNS, load balancers | When designing cloud network architectures |
| [05-SERVERLESS](05-SERVERLESS.md) | Functions, event-driven, cold starts, scaling | When building event-driven serverless applications |
| [06-COST-OPTIMIZATION](06-COST-OPTIMIZATION.md) | Reserved instances, spot, right-sizing, budgets | **Essential — controlling cloud spend** |
| [07-MULTI-CLOUD](07-MULTI-CLOUD.md) | Multi-cloud strategies, portability, abstraction | When evaluating or operating across multiple clouds |
| [08-BEST-PRACTICES](08-BEST-PRACTICES.md) | Well-Architected Framework, tagging, governance | **Essential — production checklist** |
| [09-ANTI-PATTERNS](09-ANTI-PATTERNS.md) | Common cloud mistakes and how to avoid them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured learning guide with exercises | **Start here** after the Overview |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand the difference between IaaS, PaaS, and SaaS
   - Learn the shared responsibility model
   - Explore cloud deployment models: public, private, hybrid

2. **Pick a Cloud Provider** ([01-AWS](01-AWS.md), [02-AZURE](02-AZURE.md), or [03-GCP](03-GCP.md))
   - Create a free-tier account and explore the console
   - Deploy your first virtual machine and storage bucket
   - Understand identity and access management basics

3. **Understand Networking** ([04-NETWORKING](04-NETWORKING.md))
   - Learn how VPCs, subnets, and security groups work
   - Understand public vs private networking in the cloud
   - Explore DNS and load balancing fundamentals

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured curriculum with hands-on exercises
   - Progressive skill building from basics to production

### For Experienced Engineers

1. **Review Best Practices** ([08-BEST-PRACTICES](08-BEST-PRACTICES.md))
   - Well-Architected Framework pillars across all clouds
   - Tagging strategies, governance, and compliance
   - Operational excellence for cloud infrastructure

2. **Avoid Anti-Patterns** ([09-ANTI-PATTERNS](09-ANTI-PATTERNS.md))
   - Common cloud mistakes in architecture, security, and cost
   - Pitfalls with networking, IAM, and resource management

3. **Optimize Costs** ([06-COST-OPTIMIZATION](06-COST-OPTIMIZATION.md))
   - Reserved instances, savings plans, and spot instances
   - Right-sizing, scheduling, and budget alerts
   - FinOps practices for engineering teams

4. **Explore Serverless and Multi-Cloud** ([05-SERVERLESS](05-SERVERLESS.md), [07-MULTI-CLOUD](07-MULTI-CLOUD.md))
   - Event-driven architectures and function-as-a-service
   - Multi-cloud portability, abstraction layers, and trade-offs

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      Cloud Computing                            │
│                                                                 │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐ │
│   │  Cloud Models     │  │  Cloud Providers │  │ Networking   │ │
│   │  (Service types)  │  │  (AWS/Azure/GCP) │  │ (Connectivity│ │
│   │                   │  │                  │  │  & security) │ │
│   │  - IaaS           │  │  - Compute       │  │  - VPCs      │ │
│   │  - PaaS           │  │  - Storage       │  │  - Subnets   │ │
│   │  - SaaS           │  │  - Identity      │  │  - DNS / LB  │ │
│   └──────────────────┘  └──────────────────┘  └──────────────┘ │
│                                                                 │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐ │
│   │  Serverless       │  │  Cost            │  │ Multi-Cloud  │ │
│   │  (Event-driven    │  │  Optimization    │  │ (Portability │ │
│   │   compute)        │  │  (FinOps)        │  │  & strategy) │ │
│   │                   │  │                  │  │              │ │
│   │  - Functions      │  │  - Reserved      │  │  - Abstract  │ │
│   │  - Cold starts    │  │  - Spot / Preempt│  │  - Terraform │ │
│   │  - Event triggers │  │  - Right-sizing  │  │  - Trade-offs│ │
│   └──────────────────┘  └──────────────────┘  └──────────────┘ │
│                                                                 │
│   ┌──────────────────┐  ┌──────────────────┐                   │
│   │  Best Practices   │  │  Operations      │                   │
│   │  (Well-Architected│  │  (Governance &   │                   │
│   │   Framework)      │  │   anti-patterns) │                   │
│   │                   │  │                  │                   │
│   │  - Reliability    │  │  - Tagging       │                   │
│   │  - Security       │  │  - Compliance    │                   │
│   │  - Performance    │  │  - Automation    │                   │
│   └──────────────────┘  └──────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
```

## 🔑 Key Concepts

```
Cloud Service Models
────────────────────
IaaS                → Infrastructure as a Service — virtual machines, storage, networking
PaaS                → Platform as a Service — managed runtimes, databases, middleware
SaaS                → Software as a Service — fully managed applications (email, CRM)
FaaS                → Function as a Service — event-driven serverless compute

Cloud Providers
───────────────
AWS                 → Amazon Web Services — largest market share, broadest service catalog
Azure               → Microsoft Azure — strong enterprise/hybrid integration
GCP                 → Google Cloud Platform — strengths in data, AI/ML, Kubernetes

Networking
──────────
VPC                 → Virtual Private Cloud — isolated network within a cloud provider
Subnet              → Subdivision of a VPC, typically public or private
Security Group      → Virtual firewall controlling inbound/outbound traffic
Load Balancer       → Distributes traffic across multiple instances or services

Serverless
──────────
Lambda / Functions  → Event-triggered compute that scales to zero
Cold Start          → Latency penalty when a function instance is initialized
Event Source        → Trigger that invokes a serverless function (queue, HTTP, schedule)

Cost & Governance
─────────────────
Reserved Instance   → Discounted pricing for committed usage (1–3 years)
Spot Instance       → Spare capacity at steep discounts, can be reclaimed
Right-Sizing        → Matching resource allocation to actual workload needs
Tagging             → Key-value metadata for organizing and tracking cloud resources
```

## 📋 Topics Covered

- **Foundations** — Cloud computing models, shared responsibility, deployment models, cloud-native concepts
- **AWS** — EC2, S3, Lambda, IAM, VPC, RDS, DynamoDB, CloudFormation
- **Azure** — App Service, Functions, Entra ID, Virtual Networks, Cosmos DB, Bicep
- **GCP** — Compute Engine, Cloud Run, Cloud Functions, IAM, VPC, BigQuery, Deployment Manager
- **Networking** — VPCs, subnets, peering, DNS, load balancers, CDNs, firewalls
- **Serverless** — Functions-as-a-Service, event triggers, cold starts, scaling patterns
- **Cost Optimization** — Reserved instances, spot instances, right-sizing, budgets, FinOps
- **Multi-Cloud** — Portability, abstraction layers, Terraform, trade-offs
- **Best Practices** — Well-Architected Framework, tagging, security, governance
- **Anti-Patterns** — Common cloud mistakes in architecture, security, cost, and operations

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to cloud computing?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md) then follow [LEARNING-PATH.md](LEARNING-PATH.md)

**Already familiar with cloud?** → Jump to [05-SERVERLESS.md](05-SERVERLESS.md) or [06-COST-OPTIMIZATION.md](06-COST-OPTIMIZATION.md)

**Going to production?** → Review [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) and [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [LEARNING-PATH.md](LEARNING-PATH.md) — progressive exercises from basics to production
