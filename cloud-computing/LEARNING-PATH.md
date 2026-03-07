# Cloud Computing Learning Path

A structured, self-paced training guide to mastering cloud computing — from foundational concepts and service models to production-grade architecture, cost optimization, and multi-cloud strategies. Each phase builds on the previous one, progressing from core concepts to advanced production patterns.

> **Time Estimate:** 10–14 weeks at ~5 hours/week. Adjust pace to your experience level. Engineers with prior cloud experience may complete Phases 1–2 in half the time.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Phase 1: Cloud Foundations (Weeks 1–3)](#phase-1-cloud-foundations-weeks-13)
4. [Phase 2: Core Cloud Services (Weeks 4–6)](#phase-2-core-cloud-services-weeks-46)
5. [Phase 3: Networking and Security (Weeks 7–8)](#phase-3-networking-and-security-weeks-78)
6. [Phase 4: Advanced Topics (Weeks 9–11)](#phase-4-advanced-topics-weeks-911)
7. [Phase 5: Production Readiness (Weeks 12–14)](#phase-5-production-readiness-weeks-1214)
8. [Certification Paths](#certification-paths)
9. [Recommended Resources](#recommended-resources)
10. [Version History](#version-history)

---

## Overview

### Why This Learning Path?

Cloud computing is a broad field spanning compute, storage, networking, databases, security, cost management, and more across multiple providers. Without a structured approach, learners often jump between topics without building a solid foundation.

This path provides a **progressive curriculum** that:

1. Builds foundational understanding before diving into provider-specific services
2. Balances theory with hands-on exercises
3. Covers cross-cutting concerns (security, cost, operations) throughout
4. Prepares you for real-world cloud architecture decisions

### How to Use This Guide

- **Follow the phases in order** — each phase builds on the previous one
- **Complete the exercises** — reading alone does not build cloud skills; hands-on practice is essential
- **Use the free tier** — all major providers offer free tiers sufficient for learning exercises
- **Take notes** — document your learnings, especially provider-specific gotchas

---

## Prerequisites

Before starting this learning path, you should have:

| Prerequisite | Why It Matters |
|-------------|----------------|
| Basic networking (TCP/IP, DNS, HTTP) | Cloud networking builds on these fundamentals |
| Linux command line | Most cloud interactions involve CLI tools |
| Basic scripting (Bash, Python, or similar) | Automation is central to cloud operations |
| Version control (Git) | Infrastructure as Code lives in Git repositories |

---

## Phase 1: Cloud Foundations (Weeks 1–3)

### Goals

- Understand cloud computing models (IaaS, PaaS, SaaS)
- Learn the shared responsibility model
- Navigate the console and CLI of one major provider
- Understand regions, availability zones, and edge locations

### Topics

| Topic | Resource | Time |
|-------|----------|------|
| Cloud computing models | [00-OVERVIEW.md](00-OVERVIEW.md) — What is cloud computing, IaaS/PaaS/SaaS | 3 hours |
| Cloud deployment models | [00-OVERVIEW.md](00-OVERVIEW.md) — Public, private, hybrid, multi-cloud | 2 hours |
| Shared responsibility model | [00-OVERVIEW.md](00-OVERVIEW.md) — Security boundaries | 2 hours |
| Provider console and CLI | Provider documentation — getting started guides | 4 hours |
| Account setup and IAM basics | Provider documentation — IAM fundamentals | 4 hours |

### Exercises

1. **Create a free-tier cloud account** on your chosen provider (AWS, Azure, or GCP)
2. **Set up IAM** — create a non-root admin user with MFA enabled
3. **Install and configure the CLI** — authenticate and run basic commands (list regions, list services)
4. **Explore the console** — navigate billing, IAM, and compute sections
5. **Deploy a simple VM** — launch a virtual machine, SSH into it, and terminate it

### Milestone

You can explain IaaS vs PaaS vs SaaS, you understand the shared responsibility model, and you can create and manage basic resources through both the console and CLI.

---

## Phase 2: Core Cloud Services (Weeks 4–6)

### Goals

- Understand core compute, storage, and database services
- Deploy and manage applications on cloud infrastructure
- Use object storage and block storage appropriately
- Set up managed databases

### Topics

| Topic | Resource | Time |
|-------|----------|------|
| Core AWS services | [01-AWS.md](01-AWS.md) — EC2, S3, RDS, Lambda, VPC | 5 hours |
| Core Azure services | [02-AZURE.md](02-AZURE.md) — VMs, Blob Storage, SQL, Functions | 5 hours |
| Core GCP services | [03-GCP.md](03-GCP.md) — Compute Engine, Cloud Storage, Cloud SQL | 5 hours |

> **Note:** Focus on your primary provider. Read the others for awareness, but do hands-on work on one.

### Exercises

1. **Deploy a web application** — deploy a simple web app on a VM behind a load balancer
2. **Set up object storage** — create a bucket, upload files, configure access policies and lifecycle rules
3. **Launch a managed database** — create an RDS/Azure SQL/Cloud SQL instance, connect from your app
4. **Deploy a serverless function** — create a Lambda/Azure Function/Cloud Function triggered by HTTP
5. **Configure auto-scaling** — set up an auto-scaling group that scales based on CPU utilization

### Milestone

You can deploy a multi-tier application (web server + database) using core cloud services. You understand compute, storage, and database service options.

---

## Phase 3: Networking and Security (Weeks 7–8)

### Goals

- Design and implement VPC networking with public and private subnets
- Configure security groups, NACLs, and firewalls
- Implement IAM best practices — roles, policies, least privilege
- Understand encryption at rest and in transit

### Topics

| Topic | Resource | Time |
|-------|----------|------|
| Cloud networking | [04-NETWORKING.md](04-NETWORKING.md) — VPCs, subnets, peering, DNS, load balancers | 5 hours |
| IAM deep dive | Provider IAM documentation — policies, roles, federation | 3 hours |
| Encryption and data protection | Provider security documentation — KMS, TLS, encryption | 2 hours |

### Exercises

1. **Design a VPC** — create a VPC with public and private subnets across 2 AZs
2. **Configure security groups** — implement least-privilege network access rules
3. **Set up a bastion host or SSM** — access private instances without public IPs
4. **Implement IAM roles** — create roles for EC2 instances to access S3 without access keys
5. **Enable encryption** — encrypt an S3 bucket (at rest) and enforce HTTPS (in transit)
6. **Set up VPC peering** — connect two VPCs and route traffic between them

### Milestone

You can design a secure VPC with proper network segmentation, implement IAM roles with least privilege, and enforce encryption for data at rest and in transit.

---

## Phase 4: Advanced Topics (Weeks 9–11)

### Goals

- Understand serverless computing patterns and trade-offs
- Implement cost optimization strategies
- Evaluate multi-cloud approaches and trade-offs
- Use Infrastructure as Code for all resources

### Topics

| Topic | Resource | Time |
|-------|----------|------|
| Serverless computing | [05-SERVERLESS.md](05-SERVERLESS.md) — Functions, event-driven, cold starts | 4 hours |
| Cost optimization | [06-COST-OPTIMIZATION.md](06-COST-OPTIMIZATION.md) — Pricing models, right-sizing, reservations | 4 hours |
| Multi-cloud strategies | [07-MULTI-CLOUD.md](07-MULTI-CLOUD.md) — Patterns, abstraction, trade-offs | 3 hours |
| Infrastructure as Code | IaC documentation — Terraform or CloudFormation basics | 4 hours |

### Exercises

1. **Build a serverless API** — create an API using API Gateway + Lambda (or equivalent) with a DynamoDB/Cosmos DB backend
2. **Analyze your costs** — use Cost Explorer or equivalent to analyze your free-tier usage patterns
3. **Implement lifecycle policies** — configure S3/Blob lifecycle rules to move data between storage tiers
4. **Right-size resources** — use provider recommendations to identify over-provisioned resources
5. **Define infrastructure in code** — rewrite your Phase 2 application using Terraform or CloudFormation
6. **Set up budgets** — create a budget with alerts at 50%, 80%, and 100% thresholds

### Milestone

You can build serverless applications, implement cost optimization strategies, evaluate multi-cloud decisions critically, and manage all infrastructure through code.

---

## Phase 5: Production Readiness (Weeks 12–14)

### Goals

- Apply cloud best practices across all pillars (security, reliability, cost, performance, operations)
- Identify and avoid common cloud anti-patterns
- Design production-ready architectures with proper monitoring, logging, and alerting
- Implement disaster recovery strategies

### Topics

| Topic | Resource | Time |
|-------|----------|------|
| Best practices | [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) — Well-Architected Framework, governance | 4 hours |
| Anti-patterns | [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md) — Common mistakes and how to avoid them | 3 hours |
| Monitoring and observability | Provider monitoring documentation — CloudWatch, Azure Monitor, Cloud Monitoring | 4 hours |
| Disaster recovery | Provider DR documentation — backup, replication, failover | 4 hours |

### Exercises

1. **Conduct a Well-Architected review** — evaluate your Phase 2/4 application against the Well-Architected Framework
2. **Set up comprehensive monitoring** — dashboards for the four golden signals (latency, traffic, errors, saturation)
3. **Implement centralized logging** — aggregate logs from all services into a centralized platform
4. **Create alerts** — set up alerts for error rates, latency thresholds, and budget anomalies
5. **Test disaster recovery** — simulate an AZ failure and verify your application recovers
6. **Anti-pattern audit** — review your architecture against each anti-pattern in [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md)

### Milestone

You can design, deploy, and operate production-ready cloud architectures. You can identify anti-patterns, apply best practices, and implement monitoring, alerting, and disaster recovery.

---

## Certification Paths

Cloud certifications validate your knowledge and are valued by employers. Here are recommended starting certifications:

### AWS

| Certification | Level | Focus |
|---------------|-------|-------|
| **AWS Cloud Practitioner** | Foundational | Cloud concepts, billing, security basics |
| **AWS Solutions Architect – Associate** | Associate | Architecture, best practices, service selection |
| **AWS DevOps Engineer – Professional** | Professional | CI/CD, monitoring, automation |

### Azure

| Certification | Level | Focus |
|---------------|-------|-------|
| **AZ-900: Azure Fundamentals** | Foundational | Cloud concepts, Azure services, pricing |
| **AZ-104: Azure Administrator** | Associate | Identity, governance, compute, networking, storage |
| **AZ-305: Azure Solutions Architect** | Expert | Design identity, governance, data, infrastructure |

### GCP

| Certification | Level | Focus |
|---------------|-------|-------|
| **Cloud Digital Leader** | Foundational | Cloud concepts, GCP services, business value |
| **Associate Cloud Engineer** | Associate | Deploy, monitor, and manage GCP solutions |
| **Professional Cloud Architect** | Professional | Design and manage cloud architectures |

### Recommended Order

1. Start with a **foundational** certification on your primary provider
2. Progress to an **associate** certification within 3–6 months
3. Consider a **professional/expert** certification after 1+ year of hands-on experience

---

## Recommended Resources

### Documentation

- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/)
- [Google Cloud Architecture Framework](https://cloud.google.com/architecture/framework)

### Books

- *Cloud Strategy* by Gregor Hohpe
- *The Cloud Adoption Framework* (available from each provider)
- *Designing Data-Intensive Applications* by Martin Kleppmann (for distributed systems context)

### Practice Platforms

- **AWS Free Tier** — 12 months of free-tier services
- **Azure Free Account** — 12 months of free services + $200 credit
- **GCP Free Tier** — always-free services + $300 credit for 90 days

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial version |
