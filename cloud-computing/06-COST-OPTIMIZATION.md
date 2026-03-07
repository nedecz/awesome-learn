# Cloud Cost Optimization

## Table of Contents

1. [Overview](#overview)
2. [Understanding Cloud Costs](#understanding-cloud-costs)
3. [Pricing Models](#pricing-models)
4. [Right-Sizing Resources](#right-sizing-resources)
5. [Reserved and Savings Plans](#reserved-and-savings-plans)
6. [Spot and Preemptible Instances](#spot-and-preemptible-instances)
7. [Storage Optimization](#storage-optimization)
8. [Network Cost Optimization](#network-cost-optimization)
9. [Auto-Scaling Strategies](#auto-scaling-strategies)
10. [Budgets and Alerts](#budgets-and-alerts)
11. [FinOps Practices](#finops-practices)
12. [Cost Optimization by Provider](#cost-optimization-by-provider)
13. [Prerequisites](#prerequisites)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

This documentation covers cloud cost optimization strategies, pricing models, and FinOps practices across AWS, Azure, and GCP. Cloud spending can quickly spiral out of control without deliberate cost management — understanding pricing models, right-sizing resources, and implementing governance are essential skills for any cloud practitioner.

### Target Audience

- **Engineers** provisioning and managing cloud infrastructure who need to understand cost implications of their choices
- **Architects** designing cost-efficient cloud architectures and selecting appropriate pricing models
- **Engineering Managers** responsible for cloud budgets and cost accountability within their teams
- **FinOps Practitioners** driving cost optimization culture and practices across the organization

### Scope

- Cloud pricing fundamentals: on-demand, reserved, spot, and savings plans
- Right-sizing compute, memory, and storage resources
- Auto-scaling strategies for cost and performance balance
- Storage tiering and lifecycle policies
- Network cost drivers and optimization techniques
- Budget management, alerts, and cost allocation
- FinOps framework: inform, optimize, operate
- Provider-specific cost tools and recommendations

---

## Understanding Cloud Costs

### The Cloud Cost Model

Cloud computing shifts infrastructure spending from capital expenditure (CapEx) to operational expenditure (OpEx). Instead of purchasing and depreciating physical hardware over years, organizations pay for resources as they consume them.

This model offers flexibility but introduces a new challenge: **cost is directly proportional to usage**, and without governance, usage tends to grow faster than expected.

### Cost Categories

| Category | Examples | Typical Share |
|----------|----------|---------------|
| **Compute** | VMs, containers, functions, managed compute | 50–70% |
| **Storage** | Block storage, object storage, file systems, backups | 10–20% |
| **Networking** | Data transfer, load balancers, VPN, DNS | 5–15% |
| **Databases** | Managed databases, data warehouses, caches | 10–20% |
| **Other** | Monitoring, security services, support plans | 5–10% |

### Hidden Cost Drivers

Many cloud costs are not immediately obvious:

- **Data egress charges** — transferring data out of a cloud region or between regions incurs significant fees
- **Cross-AZ traffic** — even traffic between availability zones within the same region is charged
- **Idle resources** — VMs, load balancers, and IP addresses running but unused
- **Over-provisioned storage** — volumes provisioned at peak capacity but used at a fraction
- **Unattached resources** — orphaned disks, snapshots, and elastic IPs
- **Logging and monitoring** — high-volume log ingestion and metric storage

### The 80/20 Rule of Cloud Cost

In most organizations, 80% of cloud spend comes from 20% of resources. Focus optimization efforts on:

1. **Largest compute instances** — right-size or reserve the biggest VMs first
2. **Data transfer patterns** — architect to minimize cross-region and egress traffic
3. **Database instances** — right-size and choose appropriate engine tiers
4. **Storage volumes** — implement lifecycle policies and tiering

---

## Pricing Models

### On-Demand (Pay-As-You-Go)

The default pricing model. No commitment, maximum flexibility, highest per-unit cost.

**When to use:**
- Unpredictable or spiky workloads
- Short-lived development and testing environments
- New applications before usage patterns are established
- Workloads that cannot tolerate interruption and have no predictable baseline

### Reserved Instances / Committed Use

Commit to a specific amount of compute capacity for 1 or 3 years in exchange for significant discounts.

| Provider | Product | 1-Year Discount | 3-Year Discount |
|----------|---------|-----------------|-----------------|
| **AWS** | Reserved Instances / Savings Plans | 30–40% | 50–60% |
| **Azure** | Reserved VM Instances | 30–40% | 55–65% |
| **GCP** | Committed Use Discounts | 37% | 55% |

**When to use:**
- Steady-state production workloads with predictable usage
- Databases and application servers running 24/7
- Baseline compute that is always needed regardless of load

### Spot / Preemptible Instances

Use spare cloud capacity at steep discounts (60–90% off on-demand), but instances can be reclaimed with short notice.

| Provider | Product | Typical Discount | Interruption Notice |
|----------|---------|------------------|---------------------|
| **AWS** | Spot Instances | 60–90% | 2 minutes |
| **Azure** | Spot VMs | 60–90% | 30 seconds |
| **GCP** | Preemptible / Spot VMs | 60–91% | 30 seconds |

**When to use:**
- Batch processing and data pipelines
- CI/CD build runners
- Stateless web workers behind load balancers
- Machine learning training jobs with checkpointing

### Sustained Use Discounts (GCP)

GCP automatically applies discounts when instances run for a significant portion of the month — no commitment required.

- Instances running >25% of the month receive increasing discounts
- Up to 30% discount for instances running the full month
- Applied automatically — no action needed

---

## Right-Sizing Resources

### What is Right-Sizing?

Right-sizing means matching resource allocation (CPU, memory, storage, network) to actual workload requirements. Over-provisioning wastes money; under-provisioning degrades performance.

### Right-Sizing Process

```
┌─────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  1. Collect      │───▶│  2. Analyze       │───▶│  3. Recommend    │
│  Utilization     │    │  Usage Patterns   │    │  New Sizes       │
│  Metrics         │    │                   │    │                  │
└─────────────────┘    └──────────────────┘    └──────────────────┘
                                                        │
┌─────────────────┐    ┌──────────────────┐            │
│  5. Verify       │◀───│  4. Implement     │◀───────────┘
│  Performance     │    │  Changes          │
│                  │    │                   │
└─────────────────┘    └──────────────────┘
```

### Key Metrics for Right-Sizing

| Metric | Threshold for Downsizing | Tool |
|--------|--------------------------|------|
| CPU utilization | <40% average over 14 days | CloudWatch, Azure Monitor, Cloud Monitoring |
| Memory utilization | <40% average over 14 days | Custom agent or cloud-native tools |
| Disk IOPS | <50% of provisioned IOPS | Cloud provider metrics |
| Network throughput | <30% of instance bandwidth | Cloud provider metrics |

### Right-Sizing Tools by Provider

| Provider | Native Tool | Description |
|----------|-------------|-------------|
| **AWS** | AWS Compute Optimizer | ML-based recommendations for EC2, EBS, Lambda |
| **AWS** | AWS Cost Explorer Right Sizing | Identifies underutilized EC2 instances |
| **Azure** | Azure Advisor | Cost recommendations for VMs, SQL, storage |
| **GCP** | Recommender | VM and disk right-sizing recommendations |

### Common Right-Sizing Mistakes

- **Using peak usage to size** — size for P95, not P100; use auto-scaling for spikes
- **Ignoring memory** — CPU may be low but memory may be the constraint
- **Not accounting for burst** — some instance types provide burstable CPU (e.g., AWS t3)
- **Right-sizing too aggressively** — leave 20–30% headroom for unexpected load

---

## Reserved and Savings Plans

### AWS Savings Plans

AWS offers two types of Savings Plans:

| Plan Type | Flexibility | Discount |
|-----------|-------------|----------|
| **Compute Savings Plan** | Any EC2 instance family, region, OS, or tenancy; also covers Fargate and Lambda | Lower discount |
| **EC2 Instance Savings Plan** | Specific instance family in a specific region | Higher discount |

**Best practice:** Start with Compute Savings Plans for flexibility, then layer EC2 Instance Savings Plans for the most predictable workloads.

### Azure Reservations

Azure Reserved VM Instances offer 1-year and 3-year terms:

- **Instance size flexibility** — a reservation for a D4s_v3 can apply to two D2s_v3 instances
- **Scope options** — shared across subscriptions, single subscription, or resource group
- **Exchangeable** — swap for different VM sizes or regions (within the same term)

### GCP Committed Use Discounts

GCP Committed Use Discounts (CUDs) apply to:

- **Compute Engine** — commit to vCPU and memory quantities
- **Cloud SQL** — commit to database instance sizes
- **Resource-based** — commit to specific resource quantities, not instance types

### Reservation Coverage Strategy

```
Total Compute
┌────────────────────────────────────────────┐
│                                            │
│  Reserved / Committed    On-Demand  Spot   │
│  (baseline)              (buffer)   (batch)│
│  ████████████████████    ████████   ░░░░░  │
│  60-70%                  15-25%     10-15%  │
│                                            │
└────────────────────────────────────────────┘
```

---

## Spot and Preemptible Instances

### Designing for Spot

Spot instances require fault-tolerant application design:

1. **Stateless services** — no local state that would be lost on termination
2. **Checkpointing** — save progress regularly for long-running jobs
3. **Graceful shutdown** — handle termination notices to drain connections
4. **Diversification** — spread across multiple instance types and AZs

### Spot Strategies by Workload

| Workload Type | Strategy | Notes |
|---------------|----------|-------|
| Web servers | Mixed instance policy in ASG | Combine on-demand (baseline) with spot (scale-out) |
| Batch processing | Spot fleet with checkpointing | Diversify across instance types; checkpoint every N minutes |
| CI/CD runners | Spot with fallback to on-demand | Self-hosted runners on spot; fall back to on-demand if unavailable |
| ML training | Spot with managed checkpoints | Use SageMaker managed spot or custom checkpointing |
| Data processing | Spot with EMR/Dataproc | Native spot support in managed big data services |

### Handling Interruptions

```
Spot Interruption Flow
──────────────────────

1. Cloud provider reclaims capacity
2. Instance receives termination notice (2 min AWS, 30 sec Azure/GCP)
3. Application:
   a. Stops accepting new work
   b. Completes or checkpoints in-progress tasks
   c. Deregisters from load balancer / service discovery
   d. Shuts down gracefully
4. Auto-scaling group or fleet replaces with new instance
```

---

## Storage Optimization

### Storage Tiering

All major cloud providers offer multiple storage tiers with different cost/access trade-offs:

| Tier | AWS | Azure | GCP | Use Case |
|------|-----|-------|-----|----------|
| **Hot** | S3 Standard | Blob Hot | Standard | Frequently accessed data |
| **Warm** | S3 Standard-IA | Blob Cool | Nearline | Monthly access |
| **Cold** | S3 Glacier Instant | Blob Cold | Coldline | Quarterly access |
| **Archive** | S3 Glacier Deep Archive | Blob Archive | Archive | Yearly access, compliance |

### Lifecycle Policies

Automate data movement between tiers based on age or access patterns:

```
Object uploaded → Hot (0-30 days)
                → Warm (30-90 days)
                → Cold (90-365 days)
                → Archive (365+ days)
                → Delete (per retention policy)
```

### Storage Optimization Checklist

- [ ] Implement lifecycle policies on all object storage buckets
- [ ] Delete unattached EBS volumes / managed disks
- [ ] Remove old snapshots beyond retention requirements
- [ ] Use appropriate storage classes for each workload
- [ ] Compress data before storing (especially logs and backups)
- [ ] Enable deduplication where supported
- [ ] Review and clean up unused container images in registries

---

## Network Cost Optimization

### Data Transfer Pricing

Network egress is one of the most expensive and overlooked cloud costs:

| Transfer Type | Approximate Cost |
|---------------|------------------|
| Within same AZ | Free (most providers) |
| Cross-AZ (same region) | $0.01/GB |
| Cross-region | $0.02–0.09/GB |
| Internet egress | $0.05–0.12/GB |

### Network Optimization Strategies

1. **Keep traffic in the same AZ** — colocate tightly coupled services
2. **Use private endpoints** — avoid routing through the public internet for cloud service access
3. **Implement CDNs** — cache static content at edge locations to reduce origin egress
4. **Compress data in transit** — reduce the bytes transferred between services
5. **Use VPC endpoints** — access S3, DynamoDB, and other services without NAT gateway costs
6. **Consolidate NAT gateways** — each NAT gateway has a fixed hourly cost plus data processing fees

### NAT Gateway Cost Trap

NAT gateways are a common hidden cost driver:

- **Hourly charge** — ~$0.045/hour per gateway ($32/month)
- **Data processing** — ~$0.045/GB processed
- **Solution** — Use VPC endpoints for AWS services, consolidate gateways, consider NAT instances for dev

---

## Auto-Scaling Strategies

### Scaling Policies for Cost Optimization

| Policy Type | Description | Best For |
|-------------|-------------|----------|
| **Target tracking** | Maintain a target metric (e.g., 70% CPU) | General workloads |
| **Step scaling** | Add/remove specific capacity at thresholds | Workloads with known scaling steps |
| **Scheduled scaling** | Scale based on time-of-day or day-of-week | Predictable traffic patterns |
| **Predictive scaling** | ML-based forecasting of capacity needs | Recurring patterns with lead time |

### Scale-to-Zero

For non-production environments, implement scale-to-zero during off-hours:

- **Development environments** — shut down evenings and weekends (save ~65%)
- **Staging environments** — run only during business hours or on-demand
- **Serverless** — inherently scales to zero; prefer for variable workloads

### Auto-Scaling Best Practices

- Set **scale-in cooldown** periods to prevent flapping
- Use **warm pools** (AWS) to reduce scale-out latency without keeping instances running
- Monitor **scaling events** to identify patterns and opportunities
- Combine **scheduled scaling** with reactive policies for known peaks
- Test scaling behavior with **load tests** before production deployment

---

## Budgets and Alerts

### Budget Configuration

Every cloud account should have budgets configured with alerts:

| Alert Threshold | Action |
|-----------------|--------|
| 50% of budget | Informational notification to team |
| 80% of budget | Warning to engineering leads |
| 100% of budget | Alert to management; review spending |
| Anomaly detection | Immediate investigation |

### Cost Allocation Tags

Tags are essential for attributing costs to teams, projects, and environments:

| Tag | Purpose | Example |
|-----|---------|---------|
| `Environment` | Separate prod/staging/dev costs | `production`, `staging`, `dev` |
| `Team` | Attribute costs to owning team | `platform`, `payments`, `frontend` |
| `Project` | Track costs per project/product | `checkout-v2`, `search` |
| `CostCenter` | Financial reporting alignment | `CC-1234` |

### Provider Budget Tools

| Provider | Tool | Features |
|----------|------|----------|
| **AWS** | AWS Budgets | Cost, usage, reservation budgets with SNS alerts |
| **Azure** | Cost Management + Billing | Budget alerts, cost analysis, advisor recommendations |
| **GCP** | Cloud Billing Budgets | Programmatic budgets with Pub/Sub notifications |

---

## FinOps Practices

### What is FinOps?

FinOps (Financial Operations) is a cultural practice that brings financial accountability to cloud spending. It is a cross-functional discipline combining engineering, finance, and business to make informed spending decisions.

### FinOps Framework

```
┌───────────────────────────────────────────────┐
│                 FinOps Lifecycle               │
│                                               │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│   │  Inform   │─▶│ Optimize │─▶│ Operate  │   │
│   │           │  │          │  │          │   │
│   │ Visibility│  │ Rates    │  │ Governance│  │
│   │ Allocation│  │ Usage    │  │ Automation│  │
│   │ Benchmarks│  │ Rightsiz.│  │ Policies │   │
│   └──────────┘  └──────────┘  └──────────┘   │
│        ▲                            │         │
│        └────────────────────────────┘         │
│              Continuous Iteration              │
└───────────────────────────────────────────────┘
```

### FinOps Maturity Model

| Phase | Characteristics |
|-------|-----------------|
| **Crawl** | Basic cost visibility, manual tagging, ad-hoc optimization |
| **Walk** | Automated tagging, reservation planning, regular reviews |
| **Run** | Real-time optimization, automated policies, cost-aware engineering culture |

### Key FinOps Metrics

- **Unit economics** — cost per transaction, cost per customer, cost per request
- **Reservation coverage** — percentage of eligible spend covered by reservations
- **Waste percentage** — idle and over-provisioned resources as % of total spend
- **Cost per environment** — production vs non-production spending ratio

---

## Cost Optimization by Provider

### AWS Cost Optimization

- **AWS Cost Explorer** — analyze spending trends and forecast future costs
- **AWS Compute Optimizer** — ML-based instance recommendations
- **AWS Trusted Advisor** — checks for idle resources and optimization opportunities
- **S3 Intelligent-Tiering** — automatic tier selection based on access patterns

### Azure Cost Optimization

- **Azure Cost Management** — cost analysis, budgets, and recommendations
- **Azure Advisor** — personalized best practice recommendations
- **Azure Hybrid Benefit** — use existing Windows/SQL Server licenses in Azure
- **Dev/Test pricing** — discounted rates for development and testing workloads

### GCP Cost Optimization

- **GCP Recommender** — right-sizing and idle resource recommendations
- **Sustained use discounts** — automatic discounts for consistent usage
- **Preemptible VMs** — up to 91% discount for fault-tolerant workloads
- **Active Assist** — ML-powered recommendations for cost, security, and performance

---

## Prerequisites

Before diving into cloud cost optimization, you should be familiar with:

| Prerequisite | Resource |
|-------------|----------|
| Cloud computing fundamentals | [00-OVERVIEW.md](00-OVERVIEW.md) |
| At least one cloud provider | [01-AWS.md](01-AWS.md), [02-AZURE.md](02-AZURE.md), or [03-GCP.md](03-GCP.md) |
| Cloud networking basics | [04-NETWORKING.md](04-NETWORKING.md) |

---

## Next Steps

| Next | Description |
|------|-------------|
| [07-MULTI-CLOUD.md](07-MULTI-CLOUD.md) | Multi-cloud strategies, portability, and abstraction |
| [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) | Cloud computing best practices and Well-Architected Framework |
| [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md) | Common cloud computing mistakes and how to avoid them |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial version |
