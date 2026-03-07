# DNS and Name Resolution

## Table of Contents

1. [Overview](#overview)
2. [Why DNS Matters](#why-dns-matters)
3. [How Resolution Works](#how-resolution-works)
4. [Common Record Types](#common-record-types)
5. [Caching and TTL](#caching-and-ttl)
6. [Service Discovery Patterns](#service-discovery-patterns)
7. [Troubleshooting DNS](#troubleshooting-dns)
8. [Version History](#version-history)

---

## Overview

DNS converts human-friendly names into machine-friendly data such as IP addresses, mail routes, and service aliases. It is one of the most frequent hidden causes of production incidents because failures can look like application or network timeouts.

---

## Why DNS Matters

Applications should depend on stable names rather than hard-coded IP addresses. DNS enables:

- Service migration without application rewrites
- Load balancing across multiple endpoints
- Failover between regions or providers
- Environment-specific routing through split-horizon DNS

---

## How Resolution Works

```text
Application → OS resolver → recursive resolver → root → TLD → authoritative server → answer
```

Typical steps:

1. Application asks the local resolver for `api.example.com`
2. If not cached, the resolver asks a recursive DNS server
3. The recursive resolver follows delegation until it reaches the authoritative server
4. The answer is cached according to its TTL

---

## Common Record Types

| Record | Purpose | Example |
|---|---|---|
| A | Name to IPv4 address | `api.example.com -> 203.0.113.10` |
| AAAA | Name to IPv6 address | `api.example.com -> 2001:db8::10` |
| CNAME | Alias to another name | `www -> app.edge.example.com` |
| MX | Mail routing | `example.com -> mail.example.com` |
| TXT | Arbitrary metadata | SPF, DKIM, ownership validation |
| NS | Delegation | Which servers are authoritative |
| SRV | Service location with port | Some internal discovery systems |

---

## Caching and TTL

TTL defines how long a resolver may cache a record.

Trade-offs:

- Low TTL → faster failover, more resolver traffic
- High TTL → better cache hit rate, slower cutovers

When changing records in production, remember that old answers may remain cached on clients, resolvers, and proxies until TTL expires.

---

## Service Discovery Patterns

Common patterns include:

- Public DNS for internet-facing applications
- Private DNS inside cloud VPCs/VNets
- Kubernetes service discovery through cluster DNS
- Service meshes and registries that integrate with or augment DNS

---

## Troubleshooting DNS

Use a predictable workflow:

1. Verify the queried name
2. Check which resolver is answering
3. Inspect TTL and record type
4. Compare results across regions or networks
5. Confirm application configuration and proxy caches

Useful commands: `dig`, `nslookup`, `host`, and resolver logs.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial version |
