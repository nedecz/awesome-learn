# IP Addressing and Subnetting

## Table of Contents

1. [Overview](#overview)
2. [IPv4 and IPv6](#ipv4-and-ipv6)
3. [CIDR and Subnets](#cidr-and-subnets)
4. [Private and Public Address Space](#private-and-public-address-space)
5. [NAT and Egress Patterns](#nat-and-egress-patterns)
6. [Planning Address Space](#planning-address-space)
7. [Next Steps](#next-steps)
8. [Version History](#version-history)

---

## Overview

Addressing decides who can talk to whom and how routes are defined. Weak address planning creates painful migrations later, especially in cloud environments where overlapping CIDR ranges break peering, VPNs, and hybrid networking.

---

## IPv4 and IPv6

### IPv4

IPv4 uses 32-bit addresses, commonly written as dotted decimal such as `10.0.1.15`.

### IPv6

IPv6 uses 128-bit addresses, commonly written in hexadecimal such as `2001:db8::15`. IPv6 greatly expands address space and removes the need for many NAT-heavy designs.

---

## CIDR and Subnets

CIDR notation combines a network address with a prefix length.

- `10.0.0.0/24` → 256 total addresses (254 usable in many environments)
- `10.0.0.0/16` → 65,536 total addresses
- `2001:db8::/64` → standard IPv6 subnet size in many environments

| Prefix | Approximate IPv4 Size | Typical Use |
|---|---|---|
| /32 | 1 address | Single host route |
| /30 | 4 addresses | Point-to-point links |
| /24 | 256 addresses | Small subnet |
| /20 | 4,096 addresses | Medium environment |
| /16 | 65,536 addresses | Large VPC or campus range |

A subnet defines the boundary between the network portion and host portion of an address.

---

## Private and Public Address Space

### Private IPv4 ranges

- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`

Private addresses are not routed on the public internet. They are common inside data centers, VPCs, and Kubernetes clusters.

### Public addresses

Public IP addresses are globally routable. Use them sparingly and put them behind load balancers, reverse proxies, or edge protection where possible.

---

## NAT and Egress Patterns

Network Address Translation (NAT) lets private hosts reach external networks by translating source addresses.

Common patterns:

- **Home / office router NAT** — many private devices share one public IP
- **Cloud NAT gateway** — private subnets reach the internet without exposing inbound access
- **Load balancer SNAT/DNAT** — traffic is rewritten as it passes through edge components

NAT is useful, but it can also hide the true client address if proxies and forwarding headers are misconfigured.

---

## Planning Address Space

Use address planning as an architecture task, not an afterthought.

Checklist:

- Reserve space for future environments and regions
- Avoid overlapping CIDR blocks between VPCs, offices, and VPN-connected sites
- Keep subnet purpose clear (`public`, `private-app`, `private-data`, `management`)
- Document which ranges are routable, translated, or isolated
- Prefer DNS names over hard-coded IPs in applications

```text
Example VPC Plan
10.40.0.0/16
├── 10.40.0.0/24    public-ingress-a
├── 10.40.1.0/24    public-ingress-b
├── 10.40.10.0/24   private-app-a
├── 10.40.11.0/24   private-app-b
├── 10.40.20.0/24   private-data-a
└── 10.40.21.0/24   private-data-b
```

---

## Next Steps

Continue to [03-ROUTING-AND-SWITCHING.md](03-ROUTING-AND-SWITCHING.md) to understand how packets move between local segments and across networks.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial version |
