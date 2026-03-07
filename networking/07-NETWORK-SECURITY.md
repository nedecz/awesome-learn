# Network Security

## Table of Contents

1. [Overview](#overview)
2. [Security Goals](#security-goals)
3. [Segmentation and Least Privilege](#segmentation-and-least-privilege)
4. [Firewalls, ACLs, and Security Groups](#firewalls-acls-and-security-groups)
5. [Secure Access Patterns](#secure-access-patterns)
6. [Zero Trust Networking](#zero-trust-networking)
7. [Common Risks](#common-risks)
8. [Next Steps](#next-steps)
9. [Version History](#version-history)

---

## Overview

Network security is about controlling who can talk to what, under which conditions, and with what level of trust. Good security architecture reduces blast radius, protects sensitive systems, and makes unauthorized movement harder.

---

## Security Goals

Core goals include:

- **Confidentiality** — prevent unauthorized access to data in transit
- **Integrity** — prevent tampering or spoofing
- **Availability** — preserve service access for legitimate users
- **Auditability** — make flows and policy decisions observable

These goals often compete, so engineers must balance safety with operability.

---

## Segmentation and Least Privilege

Segmentation divides networks by trust boundary or workload purpose.

Examples:
- Internet-facing edges
- Application subnets
- Data stores and management planes
- Build systems and administrative access paths

Least privilege means only the minimum required flows are allowed.

Bad pattern:
- `allow any -> any` inside the environment because it is "internal"

Better pattern:
- Explicitly permit only the protocols and destinations needed for each service

---

## Firewalls, ACLs, and Security Groups

These controls decide whether traffic is allowed.

| Control | Typical scope | Example use |
|--------|----------------|-------------|
| Host firewall | One machine | Block direct database access except from app nodes |
| Network ACL | Subnet or segment | Coarse policy at a boundary |
| Security group / policy | Workload or interface | Allow app service to reach Redis on one port |
| WAF | HTTP layer | Block malicious or abusive web requests |

Policies should be treated like code: reviewed, versioned, and periodically tested.

---

## Secure Access Patterns

Safer patterns include:

- Bastion or just-in-time administrative access
- Private endpoints for managed services
- Mutual TLS between sensitive services
- VPN or identity-aware proxy for operator access
- Service accounts and workload identity instead of shared static credentials

The goal is not merely blocking traffic, but creating auditable and revocable trust relationships.

---

## Zero Trust Networking

Zero trust assumes the network itself is not a source of trust.

Principles:
- Verify identity explicitly
- Enforce authorization close to the resource
- Minimize implicit trust based on source network location
- Continuously inspect and log access decisions

This model is especially useful in cloud, remote, and multi-environment systems where the old idea of a hard trusted perimeter no longer holds.

---

## Common Risks

- Flat internal networks with unrestricted east-west movement
- Administrative ports exposed to the public internet
- Shared bastion accounts or unmanaged SSH keys
- TLS terminated too early with cleartext traffic flowing through sensitive zones
- Forgotten firewall rules that allow legacy paths no one monitors anymore

---

## Next Steps

- Continue with [08-OBSERVABILITY-AND-TROUBLESHOOTING.md](08-OBSERVABILITY-AND-TROUBLESHOOTING.md)
- Review [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md) for security-related networking mistakes to avoid

---

## Version History

| Version | Date | Changes |
|--------|------|---------|
| 1.0 | 2026 | Initial network security guide |
