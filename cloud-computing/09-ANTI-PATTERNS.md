# Cloud Computing Anti-Patterns

A catalogue of the most common cloud computing mistakes — what they look like, why they are harmful, and exactly how to fix them. Use this document as a design review checklist, a pre-production gate, or a team learning resource.

---

## Table of Contents

- [Introduction](#introduction)
- [Anti-Patterns Summary Table](#anti-patterns-summary-table)
- [1. Lift-and-Shift Without Optimization](#1-lift-and-shift-without-optimization)
- [2. Ignoring the Shared Responsibility Model](#2-ignoring-the-shared-responsibility-model)
- [3. Over-Provisioning Everything](#3-over-provisioning-everything)
- [4. No Tagging or Cost Allocation](#4-no-tagging-or-cost-allocation)
- [5. Using Long-Lived Credentials](#5-using-long-lived-credentials)
- [6. Single Availability Zone Deployments](#6-single-availability-zone-deployments)
- [7. Manual Infrastructure Changes](#7-manual-infrastructure-changes)
- [8. Treating Cloud Like a Data Center](#8-treating-cloud-like-a-data-center)
- [9. Ignoring Data Transfer Costs](#9-ignoring-data-transfer-costs)
- [10. No Disaster Recovery Plan](#10-no-disaster-recovery-plan)
- [11. Premature Multi-Cloud](#11-premature-multi-cloud)
- [12. Neglecting Cloud-Native Logging and Monitoring](#12-neglecting-cloud-native-logging-and-monitoring)
- [Quick Reference Checklist](#quick-reference-checklist)
- [Next Steps](#next-steps)
- [Version History](#version-history)

---

## Introduction

### Why Anti-Patterns Matter

Cloud anti-patterns are recurring practices that seem reasonable at first but create significant cost, security, reliability, and operational problems over time. They persist because they represent the path of least resistance — lifting VMs as-is avoids re-architecture, using root credentials avoids the complexity of IAM, and deploying to a single AZ avoids the complexity of distributed state.

The patterns documented here represent real cloud failures seen across production systems. Each one is:

- **Seductive** — it felt like the right approach when first implemented
- **Harmful** — it creates failures that are expensive, insecure, or difficult to recover from
- **Fixable** — there is a well-understood better approach

Cloud infrastructure decisions compound over time. A poorly architected account structure costs little to fix in month one but becomes a multi-quarter migration project after two years of growth. A missing tagging policy is trivial to implement on day one but nearly impossible to retrofit across thousands of resources.

### How to Use This Document

- **Pre-deployment review** — check proposed architectures against each anti-pattern
- **Existing system audit** — use the checklist to evaluate running workloads
- **Team education** — use individual anti-patterns as discussion topics in architecture reviews

---

## Anti-Patterns Summary Table

| # | Anti-Pattern | Impact | Severity |
|---|-------------|--------|----------|
| 1 | Lift-and-shift without optimization | Cost, performance | 🔴 High |
| 2 | Ignoring shared responsibility model | Security | 🔴 High |
| 3 | Over-provisioning everything | Cost | 🟡 Medium |
| 4 | No tagging or cost allocation | Cost, governance | 🟡 Medium |
| 5 | Using long-lived credentials | Security | 🔴 High |
| 6 | Single Availability Zone deployments | Reliability | 🔴 High |
| 7 | Manual infrastructure changes | Operations, reliability | 🟡 Medium |
| 8 | Treating cloud like a data center | Cost, agility | 🟡 Medium |
| 9 | Ignoring data transfer costs | Cost | 🟡 Medium |
| 10 | No disaster recovery plan | Reliability | 🔴 High |
| 11 | Premature multi-cloud | Cost, complexity | 🟡 Medium |
| 12 | Neglecting logging and monitoring | Operations, security | 🔴 High |

---

## 1. Lift-and-Shift Without Optimization

### What It Looks Like

Migrating on-premises virtual machines directly to cloud VMs with identical configurations — same OS, same sizing, same architecture — without evaluating cloud-native alternatives.

### Why It's Harmful

- **Over-provisioned compute** — on-premises servers are sized for peak + growth; cloud instances should be right-sized
- **Missed managed services** — self-managing databases, caches, and queues when managed alternatives exist
- **Higher cost** — on-demand cloud VMs are more expensive per-unit than amortized on-premises hardware
- **No elasticity benefit** — static VM counts do not leverage cloud auto-scaling

### The Better Approach

1. **Assess before migrating** — evaluate each workload for cloud-native alternatives
2. **Right-size from day one** — use provider sizing tools to match actual utilization
3. **Adopt managed services** — replace self-managed databases, caches, and middleware
4. **Plan for refactoring** — even if you lift-and-shift initially, have a plan to optimize after migration

---

## 2. Ignoring the Shared Responsibility Model

### What It Looks Like

Assuming the cloud provider handles all security — no patching of VM operating systems, no encryption configuration, no network segmentation, no access reviews.

### Why It's Harmful

- **Unpatched vulnerabilities** — OS and application-level patches are the customer's responsibility on IaaS
- **Data breaches** — misconfigured S3 buckets, open security groups, and unencrypted databases
- **Compliance failures** — auditors expect customers to document their security controls

### The Better Approach

1. **Understand the boundary** — know exactly what the provider secures vs what you secure for each service
2. **Automate patching** — use SSM Patch Manager, Azure Update Management, or OS Login
3. **Enable encryption by default** — encrypt at rest and in transit for all services
4. **Audit regularly** — use AWS Config, Azure Policy, or GCP Security Command Center

---

## 3. Over-Provisioning Everything

### What It Looks Like

Requesting the largest instance types "just in case," provisioning 10x the storage needed, and setting auto-scaling minimums at peak capacity.

### Why It's Harmful

- **Wasted spend** — paying for resources that sit idle 80% of the time
- **False sense of capacity** — large instances mask performance problems that should be fixed in code
- **Compounding cost** — over-provisioning across dozens of services multiplies waste

### The Better Approach

1. **Start small** — begin with smaller instances and scale up based on observed metrics
2. **Use auto-scaling** — let demand drive capacity rather than pre-provisioning
3. **Review monthly** — check utilization reports and downsize underused resources
4. **Use burstable instances** — for workloads with variable CPU patterns (e.g., t3, B-series)

---

## 4. No Tagging or Cost Allocation

### What It Looks Like

Resources created without tags. No way to attribute costs to teams, projects, or environments. Monthly cloud bill is one opaque number.

### Why It's Harmful

- **No accountability** — nobody owns the cost because nobody knows what they are spending
- **Cannot optimize** — you cannot optimize what you cannot measure
- **Governance failure** — orphaned resources persist because ownership is unknown
- **Slow incident response** — cannot quickly identify which team owns an affected resource

### The Better Approach

1. **Define mandatory tags** — `Environment`, `Team`, `Service`, `CostCenter` at minimum
2. **Enforce at creation** — use cloud policies to block untagged resource creation
3. **Automate tagging** — include default tags in all IaC modules
4. **Report regularly** — share per-team cost reports weekly or monthly

---

## 5. Using Long-Lived Credentials

### What It Looks Like

Hardcoded AWS access keys in application code, Azure service principal secrets stored in environment variables, GCP service account JSON keys committed to version control.

### Why It's Harmful

- **Leaked credentials** — keys in code, config files, or CI/CD logs can be extracted and abused
- **No rotation** — long-lived keys are rarely rotated, extending the window of compromise
- **Blast radius** — a single leaked key can provide broad access to cloud resources
- **Audit difficulty** — actions taken with shared keys cannot be attributed to individuals

### The Better Approach

1. **Use IAM roles** — EC2 instance profiles, Azure Managed Identity, GCP workload identity
2. **Short-lived tokens** — STS temporary credentials, OIDC federation for CI/CD
3. **No keys in code** — use secrets managers (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager)
4. **Rotate regularly** — if keys are unavoidable, automate rotation with short expiration

---

## 6. Single Availability Zone Deployments

### What It Looks Like

All production instances, databases, and services deployed in a single Availability Zone. No redundancy if that AZ experiences an outage.

### Why It's Harmful

- **Single point of failure** — an AZ outage takes down the entire application
- **AZ outages happen** — every major provider has experienced single-AZ failures
- **Violates SLAs** — provider SLAs often require multi-AZ deployment for guaranteed uptime

### The Better Approach

1. **Deploy across at least 2 AZs** — for all production services and databases
2. **Use multi-AZ database configurations** — RDS Multi-AZ, Azure Zone-Redundant, Regional Cloud SQL
3. **Test AZ failure** — simulate AZ loss to verify failover works correctly
4. **Use load balancers** — distribute traffic across AZs automatically

---

## 7. Manual Infrastructure Changes

### What It Looks Like

Making changes through the cloud console UI — clicking through wizards to create VMs, editing security groups by hand, manually configuring load balancers.

### Why It's Harmful

- **Configuration drift** — manual changes diverge from documented/expected state
- **Not reproducible** — cannot recreate environments reliably from scratch
- **No audit trail** — who changed what, when, and why is unknown
- **Error-prone** — humans make mistakes, especially under pressure during incidents

### The Better Approach

1. **Use Infrastructure as Code** — Terraform, Pulumi, CloudFormation, or Bicep for all resources
2. **Store in version control** — all infrastructure definitions in Git with PR review
3. **Apply through CI/CD** — changes go through a pipeline, not the console
4. **Lock console access** — restrict console write access to break-glass scenarios

---

## 8. Treating Cloud Like a Data Center

### What It Looks Like

Buying reserved instances for everything on day one, running VMs 24/7 for batch jobs, sizing for peak capacity instead of auto-scaling, and maintaining traditional change advisory boards for infrastructure changes.

### Why It's Harmful

- **Loses cloud advantages** — elasticity, pay-per-use, and managed services are the reasons to use cloud
- **Higher cost** — running idle resources continuously costs more than on-demand with auto-scaling
- **Slower delivery** — heavyweight change processes negate cloud agility

### The Better Approach

1. **Embrace ephemeral infrastructure** — resources should be created and destroyed as needed
2. **Auto-scale by default** — size for average, scale for peaks
3. **Use serverless where appropriate** — pay only for actual execution time
4. **Modernize processes** — lightweight change management that matches cloud speed

---

## 9. Ignoring Data Transfer Costs

### What It Looks Like

Architectures that move large volumes of data between regions, AZs, or to the internet without considering transfer charges. Chatty microservices across AZ boundaries.

### Why It's Harmful

- **Surprising bills** — data transfer is often the most unexpected cost on a cloud bill
- **Compounding cost** — as traffic grows, transfer costs grow linearly
- **Cross-region is expensive** — $0.02–0.09/GB adds up quickly at scale

### The Better Approach

1. **Colocate services** — keep tightly coupled services in the same AZ where possible
2. **Use VPC endpoints** — avoid NAT gateway charges for cloud service access
3. **Implement CDNs** — cache content at edge locations to reduce origin egress
4. **Compress data** — reduce bytes transferred between services and to clients
5. **Monitor transfer costs** — set up alerts specifically for data transfer spending

---

## 10. No Disaster Recovery Plan

### What It Looks Like

No documented recovery process. Backups exist but have never been tested. Recovery time is unknown. No one has practiced restoring from backup.

### Why It's Harmful

- **Extended outages** — without a plan, recovery takes hours or days instead of minutes
- **Data loss** — untested backups may be corrupt, incomplete, or unrestorable
- **Business impact** — prolonged downtime directly impacts revenue and customer trust
- **Compliance risk** — many regulations require documented and tested DR plans

### The Better Approach

1. **Define RTO and RPO** — Recovery Time Objective and Recovery Point Objective for each service
2. **Automate backups** — automated, cross-region backups for all critical data
3. **Test restoration regularly** — quarterly at minimum, monthly for critical systems
4. **Document runbooks** — step-by-step recovery procedures that anyone on the team can follow
5. **Run DR drills** — practice the full recovery process, including communication and escalation

---

## 11. Premature Multi-Cloud

### What It Looks Like

Adopting multi-cloud from day one "to avoid vendor lock-in," abstracting everything to the lowest common denominator, building custom abstraction layers before understanding each provider's strengths.

### Why It's Harmful

- **Increased complexity** — managing multiple providers requires more expertise and tooling
- **Higher cost** — abstraction layers, cross-cloud networking, and split spending reduce discounts
- **Slower delivery** — every feature must work across all providers
- **Weaker capabilities** — abstractions reduce access to the most powerful provider-specific features

### The Better Approach

1. **Start single-cloud** — master one provider before adding another
2. **Adopt multi-cloud intentionally** — for specific, justified reasons (see [07-MULTI-CLOUD.md](07-MULTI-CLOUD.md))
3. **Use portable patterns** — containers, Kubernetes, and Terraform provide portability without full abstraction
4. **Evaluate switching costs honestly** — the cost of multi-cloud abstraction often exceeds the cost of migration

---

## 12. Neglecting Cloud-Native Logging and Monitoring

### What It Looks Like

No centralized logging. No alerting on service health. Relying on customer reports to discover outages. Debugging by SSH-ing into individual instances and reading local log files.

### Why It's Harmful

- **Blind operations** — you cannot fix what you cannot see
- **Slow incident response** — hours spent finding the right logs across dozens of instances
- **Security blindspots** — failed authentication attempts, unauthorized API calls go unnoticed
- **No capacity planning** — without usage metrics, scaling decisions are guesswork

### The Better Approach

1. **Centralize all logs** — ship logs to CloudWatch, Azure Monitor, or Cloud Logging from day one
2. **Monitor the four golden signals** — latency, traffic, errors, saturation
3. **Set up alerts** — alert on user-facing symptoms, not just infrastructure metrics
4. **Enable audit logging** — CloudTrail, Activity Log, or Audit Logs for all accounts
5. **Implement distributed tracing** — trace requests across service boundaries

---

## Quick Reference Checklist

- [ ] Workloads are optimized for cloud, not just lifted-and-shifted
- [ ] Shared responsibility model is documented and understood
- [ ] Resources are right-sized based on actual utilization metrics
- [ ] All resources have mandatory tags for cost allocation and ownership
- [ ] No long-lived credentials — using IAM roles and short-lived tokens
- [ ] Production workloads span at least 2 Availability Zones
- [ ] All infrastructure is defined in code and version-controlled
- [ ] Cloud-native patterns (auto-scaling, managed services, serverless) are used
- [ ] Data transfer costs are monitored and optimized
- [ ] Disaster recovery plan exists, is documented, and is tested regularly
- [ ] Multi-cloud is adopted intentionally, not prematurely
- [ ] Centralized logging, monitoring, and alerting are in place

---

## Next Steps

| Next | Description |
|------|-------------|
| [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) | Cloud computing best practices and Well-Architected Framework |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Structured learning guide from beginner to advanced |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial version |
