# Observability and Troubleshooting

## Table of Contents

1. [Overview](#overview)
2. [Troubleshooting Mindset](#troubleshooting-mindset)
3. [Key Signals](#key-signals)
4. [Essential Tools](#essential-tools)
5. [A Layered Workflow](#a-layered-workflow)
6. [Common Incident Patterns](#common-incident-patterns)
7. [Runbook Starter Checklist](#runbook-starter-checklist)
8. [Version History](#version-history)

---

## Overview

Networking incidents are often blamed on the wrong layer. Effective troubleshooting starts with evidence, follows the path of a request, and narrows the failure domain from name resolution to application response.

---

## Troubleshooting Mindset

Start with clear questions:

- Is the failure total or intermittent?
- Does it affect one client, one subnet, one region, or everyone?
- Is the issue name resolution, path reachability, transport establishment, TLS trust, or application response?
- Did anything change recently in DNS, routing, firewall policy, or deployment topology?

Avoid jumping straight to packet capture if a simple DNS lookup or route-table check would explain the issue.

---

## Key Signals

Useful signals include:

- DNS resolution time and answers
- TCP connect success rate and handshake latency
- TLS handshake failures and certificate expiry
- Request latency percentiles and error rates
- Retransmission counts, packet loss, and dropped flows
- Load balancer health-check status
- NAT gateway saturation or ephemeral port exhaustion

Application observability is stronger when these network-level signals are available alongside logs and traces.

---

## Essential Tools

| Tool | Use |
|------|-----|
| `dig` / `nslookup` | Inspect DNS answers and resolver behavior |
| `ping` | Basic reachability test where ICMP is allowed |
| `traceroute` / `tracepath` | Observe hop-by-hop path changes |
| `curl` | Exercise HTTP/TLS behavior directly |
| `ss` / `netstat` | Inspect listening and established connections |
| `tcpdump` / Wireshark | Capture packets for detailed analysis |
| Cloud flow logs | Validate whether packets were accepted, rejected, or routed |

Use the least invasive tool that can answer the question.

---

## A Layered Workflow

1. **Confirm the symptom** — what exactly is failing?
2. **Resolve the name** — does DNS return the intended destination?
3. **Check path and policy** — is there a valid route and an allow rule?
4. **Test connection setup** — does TCP/TLS complete?
5. **Inspect intermediaries** — load balancers, proxies, gateways, meshes
6. **Inspect the backend** — once traffic definitely arrives, look at the service

This keeps investigations efficient and helps teams avoid cross-layer confusion.

---

## Common Incident Patterns

- **Intermittent timeouts** — packet loss, retry amplification, or uneven load balancing
- **Works from one network but not another** — DNS split horizon, firewall boundary, or route asymmetry
- **Only new deployments fail** — health checks, missing listeners, or wrong service registration
- **High latency with low CPU** — network path distance, handshake overhead, or slow downstream ACKs
- **Everything healthy except one client** — local resolver cache, proxy config, VPN, or MTU issue

---

## Runbook Starter Checklist

- [ ] Identify affected clients and networks
- [ ] Verify DNS answer and TTL
- [ ] Check route tables and gateway targets
- [ ] Validate firewall / security policy decisions
- [ ] Confirm TCP/TLS establishment from the failing path
- [ ] Review load balancer and proxy health
- [ ] Capture packet or flow evidence only if earlier checks do not explain the issue

---

## Version History

| Version | Date | Changes |
|--------|------|---------|
| 1.0 | 2026 | Initial observability and troubleshooting guide |
