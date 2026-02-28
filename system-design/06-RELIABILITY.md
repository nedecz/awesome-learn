# Reliability

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [What Is Reliability](#what-is-reliability)
   - [Definition](#definition)
   - [Reliability vs Availability vs Durability](#reliability-vs-availability-vs-durability)
3. [Redundancy](#redundancy)
   - [Hardware Redundancy](#hardware-redundancy)
   - [Software Redundancy](#software-redundancy)
   - [Data Redundancy](#data-redundancy)
   - [N+1 Redundancy](#n1-redundancy)
4. [Failover Strategies](#failover-strategies)
   - [Active-Passive](#active-passive)
   - [Active-Active](#active-active)
   - [Warm Standby](#warm-standby)
   - [Pilot Light](#pilot-light)
   - [Strategy Comparison](#strategy-comparison)
5. [Disaster Recovery](#disaster-recovery)
   - [RPO and RTO](#rpo-and-rto)
   - [Backup Strategies](#backup-strategies)
   - [DR Tiers](#dr-tiers)
   - [Multi-Region Architecture](#multi-region-architecture)
6. [Fault Tolerance Patterns](#fault-tolerance-patterns)
   - [Circuit Breaker](#circuit-breaker)
   - [Retry with Backoff](#retry-with-backoff)
   - [Bulkhead Isolation](#bulkhead-isolation)
   - [Timeout](#timeout)
   - [Fallback](#fallback)
7. [Chaos Engineering](#chaos-engineering)
   - [Principles](#principles)
   - [Tools](#tools)
   - [Game Days](#game-days)
   - [Steady-State Hypothesis](#steady-state-hypothesis)
8. [Graceful Degradation](#graceful-degradation)
   - [Feature Flags](#feature-flags)
   - [Load Shedding](#load-shedding)
   - [Priority Queues](#priority-queues)
   - [Read-Only Mode](#read-only-mode)
9. [Health Checking and Self-Healing](#health-checking-and-self-healing)
   - [Liveness and Readiness Probes](#liveness-and-readiness-probes)
   - [Auto-Restart](#auto-restart)
   - [Auto-Scaling](#auto-scaling)
10. [Incident Management](#incident-management)
    - [Detection](#detection)
    - [Response](#response)
    - [Postmortems](#postmortems)
    - [Blameless Culture](#blameless-culture)
11. [SLAs, SLOs, and SLIs](#slas-slos-and-slis)
    - [Defining Reliability Targets](#defining-reliability-targets)
    - [Error Budgets](#error-budgets)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Reliability is a cornerstone of system design. A reliable system continues to function correctly even when components fail, traffic spikes, or entire data centers go offline. This document covers the strategies, patterns, and organizational practices that enable teams to build and operate systems users can depend on.

### Target Audience

- Engineers designing fault-tolerant distributed systems
- SREs defining reliability targets and incident response processes
- Architects evaluating redundancy, failover, and disaster recovery strategies

### Scope

- Redundancy models and failover strategies
- Disaster recovery planning with RPO/RTO analysis
- Fault tolerance patterns for distributed services
- Chaos engineering principles and practices
- Graceful degradation and self-healing mechanisms
- Incident management and reliability metrics (SLAs, SLOs, SLIs)

---

## What Is Reliability

### Definition

Reliability is the ability of a system to perform its intended function without failure for a specified period under stated conditions. A reliable system handles faults — hardware failures, software bugs, operator errors, and unexpected load — without losing data or becoming unavailable.

### Reliability vs Availability vs Durability

| Concept | Question It Answers | Example |
|---|---|---|
| **Reliability** | Does the system work correctly over time? | A payment service processes transactions without errors for months |
| **Availability** | Is the system reachable right now? | The API responds to 99.99% of requests within a calendar month |
| **Durability** | Is committed data safe from loss? | Stored objects survive even if multiple disks fail simultaneously |

```
  Relationship Between Reliability, Availability, and Durability
  ──────────────────────────────────────────────────────────────

  Reliability
  ├── Availability (system is reachable)
  │     └── Measured as uptime percentage (e.g., 99.95%)
  ├── Correctness (system produces right results)
  │     └── No data corruption, no silent failures
  └── Durability (data is not lost)
        └── Measured as probability of data loss (e.g., 99.999999999%)

  A system can be available but unreliable (returns wrong answers).
  A system can be reliable but unavailable (down for maintenance).
```

---

## Redundancy

Redundancy eliminates single points of failure by duplicating critical components. If one component fails, a redundant component takes over.

### Hardware Redundancy

| Component | Redundancy Approach | Example |
|---|---|---|
| **Power supply** | Dual PSUs per server, UPS, generator | Each server has two independent power feeds |
| **Network** | Dual NICs, redundant switches, diverse ISPs | Top-of-rack switches in a bonded pair |
| **Storage** | RAID arrays, mirrored disks | RAID 1 (mirroring) or RAID 6 (double parity) |
| **Servers** | Multiple servers behind a load balancer | 3 app servers serving the same workload |

### Software Redundancy

- **Process-level** — Run multiple instances of the same service
- **Service-level** — Deploy microservices with independent failure domains
- **Data-level** — Replicate databases across nodes with leader-follower or multi-leader setups
- **Region-level** — Deploy the full stack in multiple geographic regions

### Data Redundancy

```
  Replication Strategies
  ──────────────────────

  Synchronous Replication:
    Client ──▶ Primary ──write──▶ Replica (ACK before responding)
    ✅ Strong consistency    ❌ Higher write latency

  Asynchronous Replication:
    Client ──▶ Primary ──respond──▶ Client
                  └──write──▶ Replica (eventually)
    ✅ Lower write latency   ❌ Risk of data loss on primary failure

  Semi-Synchronous Replication:
    Client ──▶ Primary ──write──▶ Replica 1 (sync, ACK required)
                  └──write──▶ Replica 2 (async, best-effort)
    ✅ Balance of durability and latency
```

### N+1 Redundancy

N+1 redundancy means provisioning one more unit than the minimum needed to handle the load. If any single unit fails, the remaining N units can absorb the workload.

```
  N+1 Example (3+1 App Servers)
  ─────────────────────────────

  Normal operation (all 4 healthy):
    Server A: 25% load
    Server B: 25% load
    Server C: 25% load
    Server D: 25% load    ← the "+1"

  After Server C fails:
    Server A: 33% load
    Server B: 33% load
    Server D: 33% load    ← absorbs the extra load

  Capacity is maintained without degradation.
```

> **Best practice:** For critical systems, consider N+2 redundancy to survive simultaneous failures (e.g., one planned maintenance + one unexpected failure).

---

## Failover Strategies

### Active-Passive

One node handles all traffic (active). A standby node takes over if the active node fails.

```
  Active-Passive Failover
  ───────────────────────

  Normal:
    ┌──────────┐         ┌──────────┐
    │  Active  │◀─ traffic│  Client  │
    │  Node A  │         └──────────┘
    └──────────┘
    ┌──────────┐
    │ Passive  │  (idle, replicating data from A)
    │  Node B  │
    └──────────┘

  After failure of Node A:
    ┌──────────┐
    │  Node A  │  ✖ FAILED
    └──────────┘
    ┌──────────┐         ┌──────────┐
    │  Node B  │◀─ traffic│  Client  │
    │ (now     │         └──────────┘
    │  active) │
    └──────────┘
```

- ✅ Simple to implement
- ❌ Passive node is idle — wasted resources
- ❌ Failover introduces brief downtime during switchover

### Active-Active

All nodes handle traffic simultaneously. If one node fails, the remaining nodes absorb its share.

```
  Active-Active Failover
  ──────────────────────

  Normal:
    ┌──────────────┐
    │ Load Balancer │
    └──┬────────┬──┘
       │        │
       ▼        ▼
  ┌────────┐ ┌────────┐
  │ Node A │ │ Node B │    Both serve traffic
  │  50%   │ │  50%   │
  └────────┘ └────────┘

  After failure of Node A:
    ┌──────────────┐
    │ Load Balancer │
    └──────┬───────┘
           │
           ▼
      ┌────────┐
      │ Node B │    Serves 100% of traffic
      │  100%  │
      └────────┘
```

- ✅ No wasted resources — all nodes serve traffic
- ✅ No failover delay — traffic is simply redistributed
- ❌ More complex — requires data synchronization between nodes
- ❌ Conflict resolution needed for concurrent writes

### Warm Standby

A scaled-down replica of the production environment runs continuously. It receives data updates but does not serve live traffic until activated.

- ✅ Faster recovery than cold standby (minutes instead of hours)
- ❌ Higher cost than cold standby — resources are running but underutilized

### Pilot Light

Only the most critical core components (e.g., database replicas) run in the standby environment. Application servers and other components are provisioned on demand during failover.

- ✅ Lower cost than warm standby — only data-layer components run
- ❌ Slower recovery than warm standby — compute must be provisioned

### Strategy Comparison

| Strategy | Failover Time | Cost | Data Loss Risk | Complexity |
|---|---|---|---|---|
| **Active-Passive** | Seconds to minutes | Medium (idle standby) | Low (sync replication) | Low |
| **Active-Active** | Near-zero | High (full duplication) | Very low | High |
| **Warm Standby** | Minutes | Medium-High | Low | Medium |
| **Pilot Light** | Minutes to an hour | Low-Medium | Low | Medium |
| **Cold Site** | Hours | Low | Medium (backup lag) | Low |

---

## Disaster Recovery

### RPO and RTO

```
  Recovery Point Objective (RPO) and Recovery Time Objective (RTO)
  ────────────────────────────────────────────────────────────────

  ◀───── RPO ─────▶                    ◀────── RTO ──────▶
  Data loss window                     Recovery time window

  ──────────────────┼──────────────────┼──────────────────▶  time
  Last backup    Disaster           Service
  / checkpoint   occurs             restored

  RPO = How much data can you afford to lose?
  RTO = How quickly must you restore service?
```

| Metric | Question | Example Target |
|---|---|---|
| **RPO** | Maximum tolerable data loss | RPO = 1 hour → backups at least every hour |
| **RTO** | Maximum tolerable downtime | RTO = 15 minutes → automated failover required |

### Backup Strategies

| Strategy | Description | RPO Impact |
|---|---|---|
| **Full backup** | Complete copy of all data | Best RPO (point-in-time) |
| **Incremental backup** | Only changes since last backup | Good RPO, faster backups |
| **Differential backup** | Changes since last full backup | Moderate RPO, simpler restore |
| **Continuous replication** | Real-time streaming to replica | Near-zero RPO |
| **Snapshot** | Point-in-time image of storage | RPO equals snapshot frequency |

> **Best practice:** Follow the 3-2-1 rule — keep **3** copies of data, on **2** different media types, with **1** copy offsite.

### DR Tiers

| Tier | Name | Description | Typical RTO | Typical Cost |
|---|---|---|---|---|
| **Tier 0** | No DR | No recovery plan | Days to never | None |
| **Tier 1** | Cold site | Backup tapes/files stored offsite; infrastructure provisioned from scratch | 24–72 hours | Low |
| **Tier 2** | Warm standby | Scaled-down environment receiving data replication | 1–4 hours | Medium |
| **Tier 3** | Hot standby | Full replica running in parallel; automated failover | Minutes | High |
| **Tier 4** | Active-active multi-region | Traffic served from multiple regions simultaneously | Near-zero | Very high |

### Multi-Region Architecture

```
  Multi-Region Active-Active Architecture
  ────────────────────────────────────────

  ┌──────────────┐                     ┌──────────────┐
  │  Region A    │                     │  Region B    │
  │  (US-East)   │                     │  (EU-West)   │
  │              │                     │              │
  │ ┌──────────┐ │   cross-region     │ ┌──────────┐ │
  │ │   App    │ │   replication      │ │   App    │ │
  │ │ Servers  │ │◀──────────────────▶│ │ Servers  │ │
  │ └────┬─────┘ │                     │ └────┬─────┘ │
  │      │       │                     │      │       │
  │ ┌────▼─────┐ │   async / sync     │ ┌────▼─────┐ │
  │ │ Database │ │◀──────────────────▶│ │ Database │ │
  │ │ (Leader) │ │   replication      │ │ (Leader) │ │
  │ └──────────┘ │                     │ └──────────┘ │
  └──────────────┘                     └──────────────┘
         ▲                                    ▲
         │          ┌──────────────┐          │
         └──────────│  Global DNS  │──────────┘
                    │  (Route 53 / │
                    │  Traffic Mgr)│
                    └──────────────┘
                          ▲
                    ┌─────┴─────┐
                    │  Clients  │
                    └───────────┘

  DNS routes users to the nearest healthy region.
  If Region A fails, DNS shifts all traffic to Region B.
```

- ✅ Near-zero RTO and minimal RPO
- ❌ Data consistency challenges (conflict resolution, eventual consistency)
- ❌ High operational complexity and cost

---

## Fault Tolerance Patterns

### Circuit Breaker

Prevents a service from repeatedly calling a failing dependency. The circuit opens after a failure threshold is exceeded and closes again after a recovery period.

```
  States:  CLOSED ──(failures exceed threshold)──▶ OPEN
                                                      │
           CLOSED ◀──(trial succeeds)── HALF-OPEN ◀──┘
                                                  (timer expires)
```

- Use a circuit breaker on every synchronous inter-service call
- Monitor state transitions — frequent trips indicate a systemic problem

### Retry with Backoff

Automatically re-attempt failed operations with increasing delays between attempts.

```
  Exponential Backoff with Jitter:

  Attempt 1 ──✖── wait 1.0s
  Attempt 2 ──✖── wait 2.3s   (2s + random jitter)
  Attempt 3 ──✖── wait 4.7s   (4s + random jitter)
  Attempt 4 ──✔── success
```

| Parameter | Typical Value |
|---|---|
| **Max attempts** | 3–5 |
| **Initial delay** | 100ms–1s |
| **Max delay cap** | 30s–60s |
| **Backoff multiplier** | 2 |
| **Jitter** | Random 0–100% of delay |

- ✅ Retry idempotent operations (GET, PUT with idempotency key)
- ❌ Do not retry non-idempotent operations without safeguards
- ❌ Do not retry client errors (400, 401, 403, 404)

### Bulkhead Isolation

Partitions resources so that a failure in one subsystem does not exhaust resources across the entire service.

```
  ┌────────────────────────────────────────────────┐
  │               Order Service                     │
  │                                                 │
  │  ┌──────────────────┐  ┌──────────────────┐    │
  │  │ Payment Pool     │  │ Inventory Pool   │    │
  │  │ (10 connections) │  │ (10 connections) │    │
  │  │                  │  │                  │    │
  │  │ Payment service  │  │ Inventory calls  │    │
  │  │ is slow → only   │  │ are unaffected   │    │
  │  │ this pool blocks │  │                  │    │
  │  └──────────────────┘  └──────────────────┘    │
  └────────────────────────────────────────────────┘
```

### Timeout

Every external call must have a timeout. Without timeouts, a slow dependency can tie up threads and connections indefinitely, cascading failures through the system.

- ✅ Set connection timeouts shorter than read timeouts
- ✅ Propagate deadline budgets across service hops
- ❌ Never use infinite timeouts

### Fallback

When a dependency fails, return an alternative response instead of propagating the error.

| Strategy | Example |
|---|---|
| **Cached response** | Return last known product prices from cache |
| **Default value** | Recommendation service returns popular items |
| **Degraded response** | User profile without avatar |
| **Queue for later** | Accept the order, retry payment asynchronously |

---

## Chaos Engineering

### Principles

Chaos engineering is the discipline of experimenting on a system to build confidence in its ability to withstand turbulent conditions in production.

1. **Start with a steady-state hypothesis** — Define what "normal" looks like using measurable metrics
2. **Vary real-world events** — Inject failures that actually happen (server crash, network partition, disk full)
3. **Run experiments in production** — Staging environments cannot replicate production complexity
4. **Automate experiments** — Run continuously, not as one-off exercises
5. **Minimize blast radius** — Start small, expand scope as confidence grows

### Tools

| Tool | Creator | Focus Area |
|---|---|---|
| **Chaos Monkey** | Netflix | Randomly terminates VM instances in production |
| **Litmus** | CNCF | Kubernetes-native chaos engineering framework |
| **Gremlin** | Gremlin Inc. | SaaS platform for controlled fault injection |
| **Chaos Mesh** | PingCAP | Kubernetes chaos engineering with a dashboard |
| **AWS FIS** | AWS | Fault injection for AWS resources |
| **Toxiproxy** | Shopify | Simulates network conditions (latency, partitions) |

### Game Days

A game day is a planned event where teams deliberately inject failures into a system and practice their response.

```
  Game Day Process
  ────────────────

  1. PLAN        Define scope, hypothesis, and rollback plan
       │
       ▼
  2. NOTIFY      Alert stakeholders and on-call teams
       │
       ▼
  3. INJECT      Introduce the failure (e.g., kill a database node)
       │
       ▼
  4. OBSERVE     Monitor dashboards, alerts, and system behavior
       │
       ▼
  5. RECOVER     Verify the system self-heals or execute manual recovery
       │
       ▼
  6. DEBRIEF     Document findings, action items, and improvements
```

### Steady-State Hypothesis

Before running a chaos experiment, define the steady-state hypothesis — a measurable assertion about normal system behavior.

| Metric | Steady-State Hypothesis | Failure Condition |
|---|---|---|
| **Request success rate** | ≥ 99.9% of requests return 2xx | Drops below 99.5% |
| **Latency (p99)** | ≤ 200ms | Exceeds 500ms |
| **Error rate** | ≤ 0.1% | Exceeds 1% |
| **Throughput** | ≥ 1,000 req/s | Drops below 800 req/s |

---

## Graceful Degradation

When a system cannot operate at full capacity, graceful degradation ensures it continues to serve the most critical functionality rather than failing entirely.

### Feature Flags

Feature flags allow non-critical features to be disabled at runtime without redeployment.

```
  Feature Flag Degradation
  ────────────────────────

  Normal mode:
    ✅ Search           ✅ Recommendations    ✅ Reviews
    ✅ Checkout          ✅ Wish List          ✅ Chat Support

  Degraded mode (database under pressure):
    ✅ Search           ❌ Recommendations    ❌ Reviews
    ✅ Checkout          ❌ Wish List          ❌ Chat Support

  Critical features remain available; non-critical features are disabled.
```

### Load Shedding

When demand exceeds capacity, deliberately drop low-priority requests to preserve the ability to serve high-priority ones.

```
  Load Shedding Example
  ─────────────────────

  Incoming requests:  1,200 req/s
  System capacity:      800 req/s

  Without shedding:
    All 1,200 requests compete → cascading timeouts → total failure

  With shedding (priority-based):
    ┌──────────────────────────────────────────────────┐
    │ Priority 1 (checkout):    300 req/s  → ACCEPT    │
    │ Priority 2 (search):     400 req/s  → ACCEPT    │
    │ Priority 3 (browse):     100 req/s  → ACCEPT    │
    │ Priority 4 (analytics):  400 req/s  → REJECT    │
    └──────────────────────────────────────────────────┘
    Total served: 800 req/s — within capacity
```

### Priority Queues

Assign priority levels to requests so that critical operations are processed first when the system is under load.

| Priority | Category | Example Operations |
|---|---|---|
| **P0 — Critical** | Revenue-impacting | Payment processing, order placement |
| **P1 — High** | Core user experience | Search, product pages |
| **P2 — Medium** | Supporting features | Recommendations, reviews |
| **P3 — Low** | Background tasks | Analytics, report generation |

### Read-Only Mode

When the write path is degraded (e.g., database leader failure), switch to read-only mode to continue serving queries from replicas.

- ✅ Users can still browse, search, and view content
- ❌ Writes (orders, updates, posts) are queued or rejected with a clear error
- ✅ Automatically exit read-only mode when the write path recovers

---

## Health Checking and Self-Healing

### Liveness and Readiness Probes

| Probe | Question | Failure Action |
|---|---|---|
| **Liveness** | Is the process alive and not deadlocked? | Restart the container |
| **Readiness** | Can the instance handle requests? | Remove from load balancer pool |
| **Startup** | Has initialization completed? | Wait — do not restart prematurely |

```
  Kubernetes Health Check Flow
  ────────────────────────────

  Container starts
       │
       ▼
  ┌───────────────┐
  │ Startup Probe │──── Passes ────▶ Begin liveness & readiness checks
  │ /healthz/start│
  └───────┬───────┘
          │ Fails repeatedly
          ▼
    Restart container

  ┌───────────────┐                ┌────────────────┐
  │ Liveness Probe│── Fails ──────▶│ Restart Pod    │
  │ /healthz/live │                └────────────────┘
  └───────────────┘

  ┌────────────────┐               ┌────────────────────────┐
  │ Readiness Probe│── Fails ─────▶│ Remove from Service    │
  │ /healthz/ready │               │ (stop sending traffic) │
  └────────────────┘               └────────────────────────┘
```

### Auto-Restart

Container orchestrators (Kubernetes, ECS) automatically restart failed containers based on liveness probe failures or exit codes.

- **Restart policies** — Always, OnFailure, Never
- **Backoff** — Orchestrators apply exponential backoff to avoid restart loops
- **CrashLoopBackOff** — Kubernetes delays restarts when a container repeatedly crashes

### Auto-Scaling

| Scaling Type | Trigger | Response Time |
|---|---|---|
| **Horizontal Pod Autoscaler (HPA)** | CPU/memory utilization, custom metrics | 1–2 minutes |
| **Vertical Pod Autoscaler (VPA)** | Historical resource usage | Requires pod restart |
| **Cluster Autoscaler** | Pending pods due to insufficient nodes | 3–5 minutes |
| **Predictive Autoscaler** | Forecasted load based on historical patterns | Pre-scales before demand |

- ✅ Set sensible minimum and maximum replica counts
- ✅ Scale on application-level metrics (queue depth, request latency) not just CPU
- ❌ Do not scale too aggressively — rapid scaling can overwhelm downstream dependencies

---

## Incident Management

### Detection

| Method | Description | Example |
|---|---|---|
| **Monitoring and alerting** | Threshold-based or anomaly-based alerts | PagerDuty fires when error rate > 1% |
| **Synthetic monitoring** | Automated tests that simulate user journeys | Canary checks every 60 seconds |
| **Log aggregation** | Centralized search across all service logs | Kibana dashboard for 5xx errors |
| **User reports** | Customers report issues via support channels | Support tickets spike |

### Response

```
  Incident Response Timeline
  ──────────────────────────

  Alert fires
       │
       ▼
  1. ACKNOWLEDGE     On-call engineer acknowledges the alert (< 5 min)
       │
       ▼
  2. TRIAGE          Assess severity, identify blast radius
       │
       ▼
  3. MITIGATE        Apply immediate fix (rollback, failover, scale up)
       │
       ▼
  4. COMMUNICATE     Update status page and stakeholders
       │
       ▼
  5. RESOLVE         Confirm the issue is fully resolved
       │
       ▼
  6. FOLLOW UP       Schedule postmortem, file action items
```

### Postmortems

A postmortem is a structured review conducted after an incident to understand what happened and how to prevent recurrence.

| Section | Content |
|---|---|
| **Summary** | What happened, impact, and duration |
| **Timeline** | Minute-by-minute account of the incident |
| **Root cause** | The underlying technical or process failure |
| **Contributing factors** | Conditions that amplified the impact |
| **What went well** | Effective responses and systems that worked |
| **What went wrong** | Gaps in detection, response, or recovery |
| **Action items** | Concrete improvements with owners and deadlines |

### Blameless Culture

- Focus on **systems and processes**, not individuals
- Ask "what failed" instead of "who failed"
- Treat incidents as learning opportunities
- Share postmortems widely so the entire organization benefits
- Measure improvement by tracking action item completion rates

---

## SLAs, SLOs, and SLIs

### Defining Reliability Targets

```
  SLI → SLO → SLA
  ────────────────

  SLI (Service Level Indicator):
    A quantitative measure of service behavior.
    Example: "Proportion of requests completed in < 200ms"

  SLO (Service Level Objective):
    A target value for an SLI.
    Example: "99.9% of requests complete in < 200ms over a 30-day window"

  SLA (Service Level Agreement):
    A contract with consequences if the SLO is not met.
    Example: "If availability drops below 99.9%, customer receives service credits"
```

| Term | Definition | Audience |
|---|---|---|
| **SLI** | Metric that measures service behavior | Engineering |
| **SLO** | Internal reliability target for an SLI | Engineering + Product |
| **SLA** | External contract with penalties for breach | Business + Customers |

**Common SLIs:**

| SLI | Calculation |
|---|---|
| **Availability** | Successful requests / Total requests × 100 |
| **Latency** | Proportion of requests faster than a threshold (e.g., p99 < 200ms) |
| **Throughput** | Requests processed per second |
| **Error rate** | Failed requests / Total requests × 100 |
| **Durability** | Proportion of data retained without loss |

### Error Budgets

An error budget is the inverse of an SLO — the amount of allowed unreliability within a given period.

```
  Error Budget Calculation
  ────────────────────────

  SLO: 99.9% availability over 30 days

  Total minutes in 30 days:  30 × 24 × 60 = 43,200 minutes
  Allowed downtime (0.1%):   43,200 × 0.001 = 43.2 minutes

  Error budget = 43.2 minutes of downtime per 30-day window
```

| SLO | Monthly Error Budget |
|---|---|
| 99% | 7 hours 12 minutes |
| 99.9% | 43.2 minutes |
| 99.95% | 21.6 minutes |
| 99.99% | 4.32 minutes |
| 99.999% | 26 seconds |

**Using error budgets:**

- ✅ If budget is healthy → ship new features, run chaos experiments
- ❌ If budget is exhausted → freeze deployments, focus on reliability improvements
- ✅ Track burn rate — how fast the budget is being consumed
- ✅ Alert on high burn rates before the budget is fully spent

---

## Next Steps

Continue to [Common Designs](07-COMMON-DESIGNS.md) to explore practical system design examples including URL shorteners, chat systems, notification services, and rate limiters that apply the reliability principles covered here.

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-01-20 | Initial version — redundancy, failover strategies, disaster recovery, fault tolerance patterns, chaos engineering, graceful degradation, health checking, incident management, SLAs/SLOs/SLIs |
