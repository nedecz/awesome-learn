# HTTP, TLS and Load Balancing

## Table of Contents

1. [Overview](#overview)
2. [HTTP on Top of TCP](#http-on-top-of-tcp)
3. [TLS Basics](#tls-basics)
4. [Reverse Proxies and Gateways](#reverse-proxies-and-gateways)
5. [Load Balancing Strategies](#load-balancing-strategies)
6. [Operational Considerations](#operational-considerations)
7. [Version History](#version-history)

---

## Overview

Most modern application traffic is HTTP over TLS. Understanding how these layers interact helps explain connection reuse, certificate failures, timeouts, and the role of load balancers.

---

## HTTP on Top of TCP

HTTP relies on transport connections to move requests and responses.

Key ideas:

- HTTP/1.1 commonly uses keep-alive to reuse TCP connections
- HTTP/2 multiplexes many streams over one connection
- HTTP/3 uses QUIC over UDP rather than TCP

Common failure modes include connection exhaustion, head-of-line blocking, idle timeout mismatch, and proxy misconfiguration.

---

## TLS Basics

TLS provides:

- **Confidentiality** — traffic is encrypted in transit
- **Integrity** — changes to data are detectable
- **Authenticity** — certificates prove server identity

Checklist:

- Use modern protocol versions and ciphers
- Automate certificate issuance and renewal
- Monitor expiration proactively
- Validate internal TLS requirements, not just internet-facing endpoints

---

## Reverse Proxies and Gateways

A reverse proxy sits in front of services and can handle:

- TLS termination
- Request routing
- Rate limiting
- Authentication or WAF integration
- Observability and access logging

Examples include NGINX, Envoy, HAProxy, cloud application gateways, and API gateways.

---

## Load Balancing Strategies

| Strategy | How It Works | Good For |
|---|---|---|
| Round robin | Rotate requests evenly | Similar backend capacity |
| Least connections | Prefer less-busy instances | Long-lived connections |
| Weighted | Distribute proportionally by weight | Mixed instance sizes |
| Hash-based | Route by client/IP/header/key | Session affinity or shard locality |
| Health-aware failover | Remove unhealthy backends | High availability |

Health checks are critical. A load balancer can only protect users if it knows which backends are healthy.

---

## Operational Considerations

Watch for these common issues:

- TLS terminated at the edge but traffic left unencrypted internally without a conscious decision
- Misconfigured `X-Forwarded-For` or `Forwarded` headers leading to wrong client IP logging
- Idle timeout mismatch between client, load balancer, proxy, and backend
- Sticky sessions hiding state management problems in applications

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial version |
