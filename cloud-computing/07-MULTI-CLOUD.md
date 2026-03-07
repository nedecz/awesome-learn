# Multi-Cloud Strategies

## Table of Contents

1. [Overview](#overview)
2. [What is Multi-Cloud](#what-is-multi-cloud)
3. [Multi-Cloud vs Hybrid Cloud](#multi-cloud-vs-hybrid-cloud)
4. [Drivers for Multi-Cloud](#drivers-for-multi-cloud)
5. [Multi-Cloud Architecture Patterns](#multi-cloud-architecture-patterns)
6. [Abstraction Layers](#abstraction-layers)
7. [Identity and Access Management](#identity-and-access-management)
8. [Networking Across Clouds](#networking-across-clouds)
9. [Data Management](#data-management)
10. [Observability in Multi-Cloud](#observability-in-multi-cloud)
11. [Multi-Cloud Challenges](#multi-cloud-challenges)
12. [Prerequisites](#prerequisites)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## Overview

This documentation covers multi-cloud strategies, architecture patterns, and the trade-offs of running workloads across multiple cloud providers. Multi-cloud is increasingly common but adds significant complexity — understanding when and how to adopt it is critical for making sound architectural decisions.

### Target Audience

- **Architects** evaluating multi-cloud strategies and designing portable architectures
- **Engineers** building and operating systems that span multiple cloud providers
- **Engineering Managers** making strategic decisions about cloud provider selection and diversification
- **Platform Engineers** building internal platforms that abstract cloud provider differences

### Scope

- Definition and motivation for multi-cloud adoption
- Multi-cloud vs hybrid cloud distinctions
- Architecture patterns for multi-cloud deployments
- Abstraction tools and frameworks (Terraform, Kubernetes, Crossplane)
- Cross-cloud identity, networking, and data management
- Observability and operational challenges
- When multi-cloud makes sense and when it does not

---

## What is Multi-Cloud

**Multi-cloud** is the practice of using services from more than one cloud provider to run an organization's workloads. This can range from using different providers for different workloads (polycloud) to running the same workload across multiple providers simultaneously (true multi-cloud).

### Multi-Cloud Spectrum

```
Less Complex                                          More Complex
────────────────────────────────────────────────────────────────────
Single Cloud    Polycloud         Active/Passive     Active/Active
                                  Multi-Cloud        Multi-Cloud

One provider    Different         Primary provider   Same workload
for everything  providers for     with failover to   runs on multiple
                different         secondary           providers
                workloads                             simultaneously
```

### Polycloud (Most Common)

Most organizations practice **polycloud** — using each provider for its strengths:

| Provider | Strength | Example Use |
|----------|----------|-------------|
| **AWS** | Broadest service catalog, ecosystem maturity | Core infrastructure, databases |
| **Azure** | Microsoft ecosystem integration, enterprise identity | Active Directory, Office 365 integration |
| **GCP** | Data analytics, machine learning, Kubernetes (GKE) | BigQuery analytics, ML workloads |

---

## Multi-Cloud vs Hybrid Cloud

| Aspect | Multi-Cloud | Hybrid Cloud |
|--------|-------------|--------------|
| **Definition** | Multiple public cloud providers | Public cloud + on-premises infrastructure |
| **Motivation** | Avoid lock-in, best-of-breed services | Data sovereignty, legacy systems, latency |
| **Complexity** | Cross-provider networking and identity | Cloud-to-on-premises connectivity |
| **Examples** | AWS + GCP + Azure | AWS + on-premises data center |
| **Common tools** | Terraform, Kubernetes, Crossplane | Azure Arc, AWS Outposts, Anthos |

These strategies are not mutually exclusive — many organizations operate in a **hybrid multi-cloud** model with on-premises infrastructure plus multiple cloud providers.

---

## Drivers for Multi-Cloud

### Valid Reasons

1. **Best-of-breed services** — use each provider's strongest offerings (e.g., GCP BigQuery + AWS Lambda)
2. **Regulatory requirements** — data sovereignty rules may require specific providers in specific regions
3. **Mergers and acquisitions** — acquired companies may already use different providers
4. **Negotiating leverage** — ability to shift workloads gives leverage in pricing negotiations
5. **Risk mitigation** — reduce dependency on a single provider for business-critical systems

### Questionable Reasons

1. **Avoiding vendor lock-in "just in case"** — the abstraction cost often exceeds the switching cost
2. **Every team picks their own cloud** — leads to fragmented expertise and operational overhead
3. **Disaster recovery across clouds** — cross-cloud DR is extremely complex and often untested
4. **Keeping options open** — without a clear strategy, multi-cloud becomes multi-problem

### Decision Framework

```
Should you go multi-cloud?
──────────────────────────

Do you have a regulatory or compliance requirement?
  → Yes → Multi-cloud may be necessary

Are you using best-of-breed services from different providers?
  → Yes → Polycloud is a natural fit

Is your primary motivation "avoiding lock-in"?
  → Yes → Carefully evaluate abstraction costs vs switching costs

Do you have the platform engineering capacity?
  → No → Stay single-cloud until you do
```

---

## Multi-Cloud Architecture Patterns

### Pattern 1: Workload Isolation

Different workloads run on different providers. Simplest pattern with the least cross-cloud complexity.

```
┌──────────────────┐    ┌──────────────────┐
│       AWS        │    │       GCP        │
│                  │    │                  │
│  Web Application │    │  Data Analytics  │
│  API Services    │    │  ML Pipelines    │
│  Databases       │    │  BigQuery        │
│                  │    │                  │
└──────────────────┘    └──────────────────┘
         │                       │
         └───── Shared Identity ─┘
               (SSO / Federation)
```

### Pattern 2: Active/Passive Failover

Primary workload on one provider with disaster recovery on another. High cost, high complexity, but provides provider-level resilience.

### Pattern 3: Kubernetes as Abstraction

Use Kubernetes as the common runtime across providers. Applications are deployed as containers and can run on any provider's Kubernetes service.

```
┌─────────────────────────────────────────┐
│            Application Layer            │
│   (Kubernetes Deployments & Services)   │
└─────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────┐
│          Kubernetes Abstraction          │
│   (Same API across all providers)       │
└─────────────────────────────────────────┘
        │               │              │
   ┌────┴────┐    ┌────┴────┐    ┌───┴────┐
   │ AWS EKS │    │Azure AKS│    │GCP GKE │
   └─────────┘    └─────────┘    └────────┘
```

### Pattern 4: Platform Abstraction

Build an internal developer platform that abstracts cloud provider specifics behind a unified API.

---

## Abstraction Layers

### Infrastructure as Code (Terraform)

Terraform's provider model enables managing resources across multiple clouds with a single tool:

- **Consistent workflow** — `plan`, `apply`, `destroy` regardless of provider
- **State management** — single source of truth for all infrastructure
- **Limitation** — resources are still provider-specific; Terraform does not make AWS EC2 portable to Azure VMs

### Kubernetes

Kubernetes provides the strongest multi-cloud abstraction for application workloads:

| Abstraction | What It Provides |
|-------------|------------------|
| **Pods** | Consistent compute unit across providers |
| **Services** | Service discovery and load balancing |
| **Ingress** | HTTP routing (provider-specific implementation) |
| **PersistentVolumes** | Storage abstraction (provider-specific drivers) |
| **ConfigMaps/Secrets** | Configuration management |

### Crossplane

Crossplane extends Kubernetes to manage cloud infrastructure using Kubernetes-native APIs:

- Provision AWS RDS, Azure SQL, or GCP Cloud SQL using the same Kubernetes manifests
- Composite resources abstract provider differences behind a unified API
- GitOps-compatible — infrastructure defined as Kubernetes objects

### Service Mesh (Istio, Linkerd)

Service meshes provide consistent networking, security, and observability across clusters and providers:

- **mTLS** — encrypted service-to-service communication regardless of provider
- **Traffic management** — routing, retries, and circuit breaking across clusters
- **Observability** — distributed tracing and metrics across provider boundaries

---

## Identity and Access Management

### Federated Identity

Centralizing identity management is critical in multi-cloud:

| Approach | Description |
|----------|-------------|
| **SSO with IdP** | Use a central identity provider (Okta, Azure AD) federated to each cloud |
| **OIDC federation** | Kubernetes service accounts federated to cloud IAM via OIDC |
| **Workload identity** | Cloud-native workload identity mapped to a central directory |

### Best Practices

- Use a **single identity provider** as the source of truth
- Implement **RBAC consistently** across all providers
- Audit access across all clouds from a **centralized log**
- Use **short-lived credentials** — avoid long-lived API keys in any provider

---

## Networking Across Clouds

### Cross-Cloud Connectivity

| Method | Latency | Bandwidth | Cost | Use Case |
|--------|---------|-----------|------|----------|
| **Public internet** | Variable | Unlimited | Low | Non-sensitive, tolerant of latency |
| **VPN (IPsec)** | Medium | Limited by VPN throughput | Low–Medium | Encrypted connectivity, moderate throughput |
| **Dedicated interconnect** | Low | High (10–100 Gbps) | High | Production cross-cloud traffic |
| **Cloud exchange** | Low | High | Medium–High | Multi-provider connectivity via Equinix/Megaport |

### DNS Across Clouds

- Use a **single DNS provider** (e.g., Route 53, Cloudflare) as the authoritative source
- Implement **health checks** and failover routing at the DNS layer
- Keep **TTLs low** for records that may need to shift between providers

---

## Data Management

### Data Gravity

Data gravity is the concept that applications and services tend to be pulled toward the data they consume. Moving large datasets between clouds is expensive and slow.

**Implications:**
- Place compute close to data whenever possible
- Replicate only the data that is needed across clouds
- Use cloud-native data services and avoid unnecessary cross-cloud data movement

### Data Replication Strategies

| Strategy | Latency | Consistency | Cost | Use Case |
|----------|---------|-------------|------|----------|
| **Synchronous replication** | High | Strong | High | Financial transactions |
| **Asynchronous replication** | Low | Eventual | Medium | Analytics, read replicas |
| **Periodic sync** | Very high | Eventual | Low | Backup, compliance |
| **Event streaming** | Medium | Eventual | Medium | Real-time cross-cloud events |

---

## Observability in Multi-Cloud

### Unified Observability

Use provider-agnostic observability tools to get a single pane of glass:

| Layer | Provider-Agnostic Tools |
|-------|------------------------|
| **Metrics** | Prometheus, Datadog, Grafana Cloud |
| **Logging** | Elasticsearch, Splunk, Grafana Loki |
| **Tracing** | Jaeger, Zipkin, Datadog APM |
| **Standard** | OpenTelemetry (vendor-neutral instrumentation) |

### Best Practices

- Instrument all applications with **OpenTelemetry** for portable telemetry
- Use a **centralized observability platform** that aggregates data from all providers
- Standardize on **common labels and tags** across all clouds for correlation
- Monitor **cross-cloud latency and error rates** as first-class metrics

---

## Multi-Cloud Challenges

### Operational Complexity

| Challenge | Single Cloud | Multi-Cloud |
|-----------|-------------|-------------|
| Skills needed | 1 provider | N providers |
| Networking | Simple | Cross-cloud VPNs, peering, DNS |
| IAM | One model | N identity models + federation |
| Incident response | One console | Multiple consoles, tools, processes |
| Cost tracking | One bill | Multiple bills, allocation challenges |
| Compliance | One provider's certifications | Certifications across all providers |

### The Lowest Common Denominator Problem

When abstracting across clouds, you often lose access to the most powerful features of each provider. The abstraction reduces capabilities to the intersection of features available everywhere.

### Cost of Multi-Cloud

Multi-cloud is not inherently cheaper. Additional costs include:

- **Platform engineering** — building and maintaining abstraction layers
- **Cross-cloud networking** — data transfer between providers
- **Skill development** — training engineers on multiple platforms
- **Tooling** — third-party tools for multi-cloud management
- **Lost discounts** — splitting spend reduces negotiating leverage with each provider

---

## Prerequisites

Before diving into multi-cloud strategies, you should be familiar with:

| Prerequisite | Resource |
|-------------|----------|
| Cloud computing fundamentals | [00-OVERVIEW.md](00-OVERVIEW.md) |
| At least one cloud provider in depth | [01-AWS.md](01-AWS.md), [02-AZURE.md](02-AZURE.md), or [03-GCP.md](03-GCP.md) |
| Cloud networking | [04-NETWORKING.md](04-NETWORKING.md) |
| Cost optimization | [06-COST-OPTIMIZATION.md](06-COST-OPTIMIZATION.md) |

---

## Next Steps

| Next | Description |
|------|-------------|
| [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) | Cloud computing best practices and Well-Architected Framework |
| [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md) | Common cloud computing mistakes and how to avoid them |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Structured learning guide from beginner to advanced |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial version |
