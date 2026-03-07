# DNS and Name Resolution

## Table of Contents

1. [Overview](#overview)
2. [Why DNS Matters](#why-dns-matters)
3. [Common Record Types](#common-record-types)
4. [Resolution Flow](#resolution-flow)
5. [Caching and TTLs](#caching-and-ttls)
6. [Service Discovery Patterns](#service-discovery-patterns)
7. [Common Failure Modes](#common-failure-modes)
8. [Next Steps](#next-steps)
9. [Version History](#version-history)

---

## Overview

Humans prefer names. Networks forward traffic using addresses. DNS bridges the two.

DNS is one of the most common hidden dependencies in distributed systems. When it fails, applications often report vague connectivity errors even though the underlying issue is simply that a name no longer resolves correctly.

---

## Why DNS Matters

DNS affects:

- Service-to-service communication
- Failover and traffic steering
- TLS certificate hostname validation
- CDN and load balancer front doors
- Hybrid and private network service discovery

A service can be perfectly healthy and still unreachable if clients resolve the wrong IP address.

---

## Common Record Types

| Record | Purpose | Example |
|-------|---------|---------|
| A | Map name to IPv4 address | `api.example.com -> 203.0.113.10` |
| AAAA | Map name to IPv6 address | `api.example.com -> 2001:db8::10` |
| CNAME | Alias one name to another | `www -> app.example.com` |
| MX | Mail routing | `example.com` mail servers |
| TXT | Free-form metadata | SPF, verification, DMARC |
| NS | Delegation to authoritative nameservers | zone ownership |
| SRV | Service location with host/port metadata | internal service discovery |

---

## Resolution Flow

A simplified lookup flow:

1. Application asks the OS resolver for `api.example.com`
2. Resolver checks local cache and hosts file
3. If needed, resolver queries a recursive DNS server
4. Recursive server walks the hierarchy or uses its own cache
5. Result is returned to the client with a TTL

```text
Application → OS resolver → Recursive resolver → Authoritative DNS → response
```

When debugging, determine whether the failure is local cache, recursive resolver, or authoritative data.

---

## Caching and TTLs

Caching improves performance and resilience, but stale data creates confusion.

- **Low TTLs** support faster failover but increase query volume
- **High TTLs** reduce load but slow down operational changes
- Some clients and libraries cache longer than expected
- Negative responses can also be cached

DNS changes that look correct in the control plane may still fail in production until caches expire.

---

## Service Discovery Patterns

Common patterns include:

- **Static hostnames** for stable services
- **Load balancer names** that abstract changing backends
- **Internal private zones** for service-to-service resolution
- **Kubernetes service DNS** for cluster-local discovery
- **Service mesh / registry integrations** for dynamic environments

The right choice depends on how quickly instances change and how much control you need over health-aware routing.

---

## Common Failure Modes

- Typo or wrong environment suffix in the hostname
- CNAME chain pointing to a deleted record
- Split-horizon DNS returning different answers than expected
- TTL set too high before a failover event
- Clients bypassing the intended resolver
- Certificates issued for a different name than the one clients use

---

## Next Steps

- Continue with [05-HTTP-HTTPS-AND-TLS.md](05-HTTP-HTTPS-AND-TLS.md) to see how successful name resolution turns into application traffic
- Use [08-OBSERVABILITY-AND-TROUBLESHOOTING.md](08-OBSERVABILITY-AND-TROUBLESHOOTING.md) for practical diagnosis techniques

---

## Version History

| Version | Date | Changes |
|--------|------|---------|
| 1.0 | 2026 | Initial DNS and name resolution guide |
