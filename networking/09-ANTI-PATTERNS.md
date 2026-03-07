# Networking Anti-Patterns

A catalogue of common networking mistakes that cause outages, confusing symptoms, and fragile systems.

---

## Table of Contents

- [Introduction](#introduction)
- [Anti-Patterns Summary Table](#anti-patterns-summary-table)
- [1. Flat, Over-Trusted Networks](#1-flat-over-trusted-networks)
- [2. Hard-Coding IP Addresses in Applications](#2-hard-coding-ip-addresses-in-applications)
- [3. Overlapping CIDR Ranges Everywhere](#3-overlapping-cidr-ranges-everywhere)
- [4. Treating DNS as an Afterthought](#4-treating-dns-as-an-afterthought)
- [5. TLS Only at the Edge by Accident](#5-tls-only-at-the-edge-by-accident)
- [6. Guess-Driven Troubleshooting](#6-guess-driven-troubleshooting)
- [7. Ignoring Timeouts and Retries](#7-ignoring-timeouts-and-retries)
- [Quick Reference Checklist](#quick-reference-checklist)
- [Next Steps](#next-steps)
- [Version History](#version-history)

---

## Introduction

Networking anti-patterns are dangerous because they often look convenient in the short term. A flat network is easier to set up, hard-coded IPs feel predictable, and broad firewall rules make things "just work." The long-term result is usually the opposite: brittle systems, larger blast radius, and incidents that take too long to diagnose.

---

## Anti-Patterns Summary Table

| # | Anti-Pattern | Severity | Impact |
|---|---|---|---|
| 1 | Flat, Over-Trusted Networks | 🔴 Critical | Large blast radius, lateral movement, hard-to-reason access |
| 2 | Hard-Coding IP Addresses in Applications | 🟠 High | Painful migrations, outages during failover, brittle deployments |
| 3 | Overlapping CIDR Ranges Everywhere | 🔴 Critical | Broken peering, VPN conflicts, impossible routing shortcuts |
| 4 | Treating DNS as an Afterthought | 🟠 High | Slow cutovers, stale traffic, confusing partial outages |
| 5 | TLS Only at the Edge by Accident | 🟠 High | Unintended plaintext traffic and inconsistent trust boundaries |
| 6 | Guess-Driven Troubleshooting | 🟠 High | Long outages, wasted effort, misdiagnosis |
| 7 | Ignoring Timeouts and Retries | 🟠 High | Retry storms, cascading failures, hidden network saturation |

---

## 1. Flat, Over-Trusted Networks

**The anti-pattern:** putting many unrelated systems in the same broad trust zone with permissive access.

**Why it hurts:** a compromise or configuration mistake spreads easily; debugging becomes harder because everything can talk to everything.

**The fix:** segment by function and trust level, then apply least-privilege rules between segments.

---

## 2. Hard-Coding IP Addresses in Applications

**The anti-pattern:** embedding IPs in application config or source code instead of depending on DNS or service discovery.

**Why it hurts:** infrastructure changes become application outages.

**The fix:** use stable names, health-aware discovery, and short-lived configuration values when IP pinning is unavoidable.

---

## 3. Overlapping CIDR Ranges Everywhere

**The anti-pattern:** independently choosing private ranges until hybrid connectivity or peering becomes necessary.

**Why it hurts:** overlapping ranges break routing and force expensive renumbering projects.

**The fix:** create an address plan early and reserve space by region, environment, and platform.

---

## 4. Treating DNS as an Afterthought

**The anti-pattern:** making DNS changes without thinking about ownership, TTL, caching, or rollout timing.

**Why it hurts:** users continue hitting old endpoints, failovers are slow, and symptoms vary by resolver cache.

**The fix:** manage DNS as production infrastructure with clear ownership, low-risk change windows, and observability.

---

## 5. TLS Only at the Edge by Accident

**The anti-pattern:** terminating TLS at a load balancer and then leaving internal traffic unencrypted without an explicit security decision.

**Why it hurts:** teams assume data is protected end to end when it is not.

**The fix:** document where TLS terminates, why, and what compensating controls exist.

---

## 6. Guess-Driven Troubleshooting

**The anti-pattern:** restarting components randomly instead of verifying DNS, routes, policy, transport, and application behavior in order.

**Why it hurts:** it extends outages and destroys evidence.

**The fix:** use a consistent troubleshooting workflow backed by logs, metrics, and packet/path inspection.

---

## 7. Ignoring Timeouts and Retries

**The anti-pattern:** allowing clients, proxies, and backends to use incompatible timeout and retry settings.

**Why it hurts:** transient failures turn into retry storms and queueing cascades.

**The fix:** design timeout budgets intentionally and document retry ownership at each layer.

---

## Quick Reference Checklist

- [ ] No broad trust zone exists without a clear reason
- [ ] Services use DNS or service discovery instead of pinned IPs
- [ ] Address ranges are centrally planned and documented
- [ ] DNS changes are treated like production changes
- [ ] TLS boundaries are explicit and reviewed
- [ ] Troubleshooting follows a repeatable workflow
- [ ] Timeout and retry policies are consistent across layers

---

## Next Steps

Review [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) and then work through [LEARNING-PATH.md](LEARNING-PATH.md) to turn these ideas into hands-on skill.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial version |
