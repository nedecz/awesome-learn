# Networking Learning Path

A structured, self-paced guide to building practical networking knowledge for software engineers, platform teams, and operators.

> **Time Estimate:** 6–8 weeks at ~4–5 hours/week.

---

## How to Use This Guide

1. Follow the phases in order — each phase builds on the previous one
2. Read the linked documents and take notes in your own words
3. Practice with real commands: `ping`, `dig`, `curl`, `ss`, and packet capture tools
4. Apply the concepts to a system you already know: a home lab, cloud VPC, Kubernetes cluster, or application stack
5. Revisit the troubleshooting phase regularly; operational skill fades without repetition

---

## Phase 1: Fundamentals (Week 1)

### Learning Objectives

- Understand packets, addresses, ports, and protocols
- Learn how latency, bandwidth, and reliability differ
- Build intuition for how requests travel between systems

### Reading

| Order | Document | Focus Areas |
|---|---|---|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | Packet flow, protocols, common failure modes |
| 2 | [01-OSI-AND-TCP-IP](01-OSI-AND-TCP-IP.md) | Layering, encapsulation, troubleshooting boundaries |

### Exercises

- Explain a web request from browser to backend in your own words
- List which protocols are involved from link layer through application layer
- Capture a simple diagram of one system you work on today

---

## Phase 2: Addressing and Reachability (Week 2)

### Learning Objectives

- Read CIDR notation confidently
- Understand private/public address space and NAT
- Learn how gateways and route tables affect reachability

### Reading

| Order | Document | Focus Areas |
|---|---|---|
| 1 | [02-IP-ADDRESSING-AND-SUBNETTING](02-IP-ADDRESSING-AND-SUBNETTING.md) | CIDR, subnetting, address planning |
| 2 | [03-ROUTING-AND-SWITCHING](03-ROUTING-AND-SWITCHING.md) | Route tables, gateways, segmentation |

### Exercises

- Design a simple `/16` network plan with separate subnets for ingress, apps, and data
- Identify the default gateway on a local or cloud host
- Explain the difference between a local subnet and a routed destination

---

## Phase 3: Names, Web Traffic, and Delivery (Week 3–4)

### Learning Objectives

- Understand how DNS resolution works end to end
- Learn how HTTP, TLS, proxies, and load balancers fit together
- Recognize common web traffic failure points

### Reading

| Order | Document | Focus Areas |
|---|---|---|
| 1 | [04-DNS-AND-NAME-RESOLUTION](04-DNS-AND-NAME-RESOLUTION.md) | Recursive resolution, TTL, record types |
| 2 | [05-HTTP-TLS-AND-LOAD-BALANCING](05-HTTP-TLS-AND-LOAD-BALANCING.md) | HTTP, TLS, health checks, reverse proxies |

### Exercises

- Use `dig` to inspect A, AAAA, CNAME, and TXT records for a domain you control or can query safely
- Trace the path of an HTTPS request through a load balancer to a backend service
- Compare how HTTP/1.1 and HTTP/2 reuse connections

---

## Phase 4: Security and Trust Boundaries (Week 5)

### Learning Objectives

- Learn how segmentation and least privilege limit blast radius
- Understand where VPNs, bastions, and zero trust fit
- Review how network policy interacts with application security

### Reading

| Order | Document | Focus Areas |
|---|---|---|
| 1 | [06-NETWORK-SECURITY](06-NETWORK-SECURITY.md) | Segmentation, firewalls, zero trust |
| 2 | [08-BEST-PRACTICES](08-BEST-PRACTICES.md) | Production-grade design and controls |

### Exercises

- Draw trust zones for one system you use today
- Identify one overly broad access rule and propose a narrower replacement
- Document where TLS terminates in your environment

---

## Phase 5: Troubleshooting and Production Operations (Week 6)

### Learning Objectives

- Build a repeatable incident workflow
- Know which networking signals and tools to use first
- Avoid guess-driven debugging

### Reading

| Order | Document | Focus Areas |
|---|---|---|
| 1 | [07-TROUBLESHOOTING-AND-OBSERVABILITY](07-TROUBLESHOOTING-AND-OBSERVABILITY.md) | Workflow, tools, and scenarios |
| 2 | [09-ANTI-PATTERNS](09-ANTI-PATTERNS.md) | Common mistakes that create incidents |

### Exercises

- Reproduce a simple failure: wrong DNS entry, blocked port, or invalid certificate in a safe environment
- Write a short runbook describing how you would diagnose a connection timeout
- Compare symptoms of DNS failure vs transport failure vs application failure

---

## Capstone Project

Pick one environment — local lab, cloud VPC, or container platform — and produce a short networking review:

- Diagram of the path from client to service to data layer
- Address plan and trust zones
- DNS and certificate ownership notes
- Load balancing and health-check strategy
- One-page troubleshooting checklist

---

## Completion Criteria

You are ready to move on when you can:

- Explain a request path clearly from DNS through application response
- Read CIDR ranges and route tables without hesitation
- Diagnose whether a failure is most likely DNS, policy, transport, or application related
- Describe at least one improvement you would make to a real network design you work with

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial version |
