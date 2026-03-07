# Networking Learning Path

A structured, self-paced guide to learning networking for software delivery — from protocol fundamentals and IP addressing to traffic management, security, and production troubleshooting.

> **Time Estimate:** 6–8 weeks at ~4–5 hours/week. Shorten or extend based on your prior exposure to networking and distributed systems.

---

## How to Use This Guide

1. **Follow the phases in order** — later topics build on earlier mental models
2. **Read the linked documents** — use them as reference material during labs and incidents
3. **Do the exercises** — networking knowledge becomes durable when you test it hands-on
4. **Tie concepts to your systems** — map what you learn onto real applications, environments, and failures
5. **Revisit the troubleshooting phase often** — it turns theory into production intuition

---

## Phase 1: Foundations (Week 1)

### Learning Objectives

- Understand packets, ports, latency, and protocol layering
- Build a clear OSI/TCP-IP mental model
- Learn the vocabulary used in networking conversations and incident reviews

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | Packets, protocols, latency, reliability, failure domains |
| 2 | [01-OSI-AND-TCP-IP](01-OSI-AND-TCP-IP.md) | Layering, encapsulation, troubleshooting by layer |

### Exercises

1. Pick one API request from your current system and map it across the layers from DNS to HTTP response.
2. List five common production error messages from your stack and identify which network layer they most likely relate to.

### Knowledge Check

- [ ] Can you explain the difference between IP, TCP, TLS, and HTTP?
- [ ] Do you know why layering speeds up troubleshooting?

---

## Phase 2: Addressing and Traffic Flow (Week 2–3)

### Learning Objectives

- Read and reason about CIDR notation
- Understand the role of default gateways and route tables
- Learn how local switching differs from inter-network routing

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-IP-ADDRESSING-AND-SUBNETTING](02-IP-ADDRESSING-AND-SUBNETTING.md) | CIDR, IPv4/IPv6, subnets, gateways, NAT |
| 2 | [03-ROUTING-AND-SWITCHING](03-ROUTING-AND-SWITCHING.md) | Switching, routers, next hops, route tables |

### Exercises

1. Sketch the subnets, gateways, and trust boundaries of an environment you know.
2. Review one cloud route table and explain where internet-bound, private, and peered traffic goes.

### Knowledge Check

- [ ] Can you determine whether two IPs are in the same subnet?
- [ ] Do you understand what the default route does?

---

## Phase 3: Name Resolution and Application Networking (Week 4)

### Learning Objectives

- Understand how names resolve to addresses
- Learn how HTTPS depends on DNS, TCP, and TLS together
- Recognize common certificate and proxy issues

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [04-DNS-AND-NAME-RESOLUTION](04-DNS-AND-NAME-RESOLUTION.md) | DNS records, recursive resolution, TTLs |
| 2 | [05-HTTP-HTTPS-AND-TLS](05-HTTP-HTTPS-AND-TLS.md) | Request lifecycle, handshakes, reverse proxies |

### Exercises

1. Use `dig` or your platform DNS tools to trace how one service name resolves in dev and prod.
2. Inspect the certificate chain of one HTTPS endpoint your team owns and verify SANs, issuer, and expiry.

### Knowledge Check

- [ ] Can you explain why a DNS issue can look like an application outage?
- [ ] Do you know the difference between TCP success and HTTP success?

---

## Phase 4: Traffic Management and Security (Week 5–6)

### Learning Objectives

- Understand L4 vs L7 load balancing
- Learn how retries, timeouts, and rate limits interact with reliability
- Apply segmentation and least privilege to network design

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [06-LOAD-BALANCING-AND-TRAFFIC-MANAGEMENT](06-LOAD-BALANCING-AND-TRAFFIC-MANAGEMENT.md) | Load balancing, health checks, failover, traffic shaping |
| 2 | [07-NETWORK-SECURITY](07-NETWORK-SECURITY.md) | Segmentation, ACLs, zero trust, secure access |
| 3 | [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Production-ready networking guidance |

### Exercises

1. Review a load balancer or ingress configuration and identify health-check, timeout, and retry behavior.
2. Audit one service and document exactly which inbound and outbound ports it requires.

### Knowledge Check

- [ ] Can you explain when L4 balancing is enough and when L7 is necessary?
- [ ] Do your systems follow least privilege for network access?

---

## Phase 5: Troubleshooting and Operational Excellence (Week 7–8)

### Learning Objectives

- Build a repeatable workflow for connectivity incidents
- Learn which tools answer which questions fastest
- Recognize common networking anti-patterns before they cause outages

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [08-OBSERVABILITY-AND-TROUBLESHOOTING](08-OBSERVABILITY-AND-TROUBLESHOOTING.md) | Signals, tools, layered workflow |
| 2 | [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | Avoidable networking mistakes |

### Exercises

1. Write a one-page runbook for a “service timeout” incident that starts at DNS and ends at backend verification.
2. Review a past incident and identify which networking signal would have shortened time to resolution.

### Knowledge Check

- [ ] Can you choose between `dig`, `curl`, `traceroute`, and packet capture based on the question at hand?
- [ ] Can you explain at least three networking anti-patterns present or previously present in your environment?

---

## Capstone Project

Document the network path for a real production or staging service:

- Entry point (CDN, load balancer, gateway)
- DNS records and TTL choices
- Subnets and route boundaries
- Required firewall / security policy flows
- TLS termination points
- Key observability signals and troubleshooting commands
- Known failure modes and failover behavior

The result should be a diagram and a short operational guide that another engineer could use during an incident.

---

## Suggested Companion Topics

Networking concepts appear throughout the repository. Pair this topic with:

- [Cloud Computing](../cloud-computing/README.md) for VPCs, peering, and cloud load balancing
- [Docker](../docker/README.md) for container networking basics
- [Kubernetes](../kubernetes/README.md) for Services, ingress, and cluster networking
- [Security](../security/README.md) for broader identity and defensive design concepts
- [Observability](../observability/README.md) for signals and incident response practices
