# Troubleshooting and Observability

## Table of Contents

1. [Overview](#overview)
2. [A Practical Troubleshooting Workflow](#a-practical-troubleshooting-workflow)
3. [Essential Tools](#essential-tools)
4. [Signals to Watch](#signals-to-watch)
5. [Common Scenarios](#common-scenarios)
6. [Incident Notes and Runbooks](#incident-notes-and-runbooks)
7. [Version History](#version-history)

---

## Overview

Networking incidents become manageable when teams use a repeatable workflow instead of guessing. Observability shortens the path from symptom to root cause by giving you packet, path, and performance evidence.

---

## A Practical Troubleshooting Workflow

1. **Define the failing path** — from which client to which service on which port?
2. **Check name resolution** — does the hostname resolve to the expected destination?
3. **Check reachability** — can packets reach the destination network and host?
4. **Check transport** — does the target port accept connections?
5. **Check policy** — are security groups, firewalls, or proxies blocking traffic?
6. **Check application behavior** — is the service healthy and returning valid responses?

This sequence avoids jumping straight to packet captures when the issue is simply DNS or a blocked port.

---

## Essential Tools

| Tool | Use |
|---|---|
| `ping` | Basic ICMP reachability and packet loss checks |
| `traceroute` / `tracepath` | Reveal path hops and where latency or drops begin |
| `dig` / `nslookup` | Inspect DNS records and resolver behavior |
| `curl` | Test HTTP endpoints, TLS, headers, and latency |
| `ss` / `netstat` | Inspect listening and established sockets |
| `tcpdump` / Wireshark | Capture and inspect packets |
| Flow logs | Observe accepted/denied traffic at network boundaries |
| Proxy / LB logs | Correlate client requests with backend behavior |

---

## Signals to Watch

- Latency by path, region, and dependency
- TCP reset counts and retransmissions
- DNS lookup latency and failure rate
- Load balancer target health
- Firewall deny counts and unexpected policy hits
- NAT gateway or egress saturation
- Packet loss and interface errors

---

## Common Scenarios

| Symptom | Investigate First |
|---|---|
| `502` or `503` at the edge | Load balancer health checks, backend listener, proxy errors |
| Connection timeout | Route table, security rule, listener, or downstream saturation |
| Hostname resolves differently by environment | Split-horizon DNS, stale caches, wrong resolver |
| Intermittent slow requests | TLS renegotiation, overloaded backend, packet loss, path change |
| Only large requests fail | MTU / fragmentation, proxy body limits, upstream timeouts |

---

## Incident Notes and Runbooks

Keep runbooks lightweight but specific:

- Known dependencies and ports
- Expected DNS names and health endpoints
- Ownership for each network segment or edge component
- Commands that have proven useful in past incidents
- Escalation points for cloud networking, security, or DNS providers

Good runbooks reduce stress and prevent repeated guesswork.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial version |
