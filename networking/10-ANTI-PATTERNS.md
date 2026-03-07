# Networking Anti-Patterns

## Table of Contents

1. [Introduction](#introduction)
2. [Anti-Patterns Summary Table](#anti-patterns-summary-table)
3. [1. Flat Internal Networks](#1-flat-internal-networks)
4. [2. Overlapping CIDR Ranges](#2-overlapping-cidr-ranges)
5. [3. DNS as an Afterthought](#3-dns-as-an-afterthought)
6. [4. Blind Retries Everywhere](#4-blind-retries-everywhere)
7. [5. Exposing Administrative Interfaces](#5-exposing-administrative-interfaces)
8. [6. Trusting the Internal Network Implicitly](#6-trusting-the-internal-network-implicitly)
9. [7. Ignoring Connection Lifecycle Limits](#7-ignoring-connection-lifecycle-limits)
10. [8. Debugging Only at the Application Layer](#8-debugging-only-at-the-application-layer)
11. [9. Unverified Failover Plans](#9-unverified-failover-plans)
12. [10. Undocumented Traffic Paths](#10-undocumented-traffic-paths)
13. [Quick Reference Checklist](#quick-reference-checklist)
14. [Version History](#version-history)

---

## Introduction

Networking anti-patterns are costly because they often remain invisible until traffic scales up, security boundaries are tested, or an incident forces engineers to understand a path no one has documented.

---

## Anti-Patterns Summary Table

| # | Anti-Pattern | Category | Severity |
|---|--------------|----------|----------|
| 1 | Flat Internal Networks | Security / Architecture | 🔴 Critical |
| 2 | Overlapping CIDR Ranges | Addressing / Routing | 🔴 Critical |
| 3 | DNS as an Afterthought | Service Discovery | 🟠 High |
| 4 | Blind Retries Everywhere | Reliability / Traffic | 🔴 Critical |
| 5 | Exposing Administrative Interfaces | Security | 🔴 Critical |
| 6 | Trusting the Internal Network Implicitly | Security | 🔴 Critical |
| 7 | Ignoring Connection Lifecycle Limits | Performance | 🟠 High |
| 8 | Debugging Only at the Application Layer | Operations | 🟠 High |
| 9 | Unverified Failover Plans | Resilience | 🟠 High |
| 10 | Undocumented Traffic Paths | Operations | 🟠 High |

---

## 1. Flat Internal Networks

Putting every workload into the same trusted segment makes lateral movement easy and policy reasoning nearly impossible. Segment by trust boundary and service purpose.

## 2. Overlapping CIDR Ranges

Overlapping address space breaks peering, VPNs, mergers, and hybrid connectivity. Reserve address space strategically before the environment grows.

## 3. DNS as an Afterthought

Changing records without thinking about TTLs, certificate names, and cache behavior causes painful incidents. Treat DNS as critical production infrastructure.

## 4. Blind Retries Everywhere

Retries without budgets, backoff, or idempotency checks amplify outages and overload already struggling backends. Retry deliberately, not reflexively.

## 5. Exposing Administrative Interfaces

SSH, database consoles, dashboards, and internal APIs should not be broadly internet-accessible. Use bastions, identity-aware access, and private connectivity.

## 6. Trusting the Internal Network Implicitly

"It is inside the VPC" is not a security strategy. Internal compromise is common enough that identity and authorization still matter inside the network.

## 7. Ignoring Connection Lifecycle Limits

Ephemeral port exhaustion, too-short idle timeouts, and undersized connection pools create failures that look random until traffic rises. Model and monitor connection behavior.

## 8. Debugging Only at the Application Layer

Application logs matter, but they cannot explain a wrong route or dropped SYN packet. Work through DNS, routing, policy, and transport before concluding the network is fine.

## 9. Unverified Failover Plans

Many teams discover during an outage that their secondary route, backup DNS target, or standby balancer has never been exercised. Test failover safely and regularly.

## 10. Undocumented Traffic Paths

If nobody knows which gateways, proxies, or policies a request depends on, incident response slows dramatically. Document the path and the owning teams.

---

## Quick Reference Checklist

- [ ] Are trust zones segmented meaningfully?
- [ ] Do CIDR plans avoid overlap across current and future environments?
- [ ] Are DNS changes coordinated with TTL, cert, and failover expectations?
- [ ] Do retries and timeouts have budgets and backoff?
- [ ] Are administrative interfaces private and audited?
- [ ] Can engineers explain the full path from client to backend?

---

## Version History

| Version | Date | Changes |
|--------|------|---------|
| 1.0 | 2026 | Initial networking anti-patterns guide |
