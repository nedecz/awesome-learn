# IP Addressing and Subnetting

## Table of Contents

1. [Overview](#overview)
2. [IPv4 and IPv6](#ipv4-and-ipv6)
3. [CIDR Notation](#cidr-notation)
4. [Subnets and Gateways](#subnets-and-gateways)
5. [Private Address Space and NAT](#private-address-space-and-nat)
6. [Subnet Design Guidelines](#subnet-design-guidelines)
7. [Common Mistakes](#common-mistakes)
8. [Next Steps](#next-steps)
9. [Version History](#version-history)

---

## Overview

Every networked system needs a way to identify endpoints and decide whether traffic is local or must be forwarded elsewhere. IP addressing and subnetting provide that structure.

---

## IPv4 and IPv6

### IPv4

IPv4 addresses are 32 bits long and usually written in dotted decimal notation, such as `10.20.30.40`.

Benefits:
- Familiar and widely supported
- Smaller headers and simpler operational habits

Limitations:
- Limited address space
- Frequent reliance on NAT in private environments

### IPv6

IPv6 addresses are 128 bits long and written in hexadecimal groups, such as `2001:db8::10`.

Benefits:
- Vast address space
- Better support for end-to-end addressing
- Modern features for auto-configuration and neighbor discovery

Operational reality: most environments are dual-stack or IPv4-heavy, so engineers should be comfortable with both.

---

## CIDR Notation

CIDR notation describes both an address and the size of its prefix:

- `192.168.1.0/24` → first 24 bits are the network portion
- `10.0.0.0/16` → larger range with more host addresses
- `2001:db8:abcd::/48` → common IPv6 prefix example

### Quick Reference

| Prefix | Approx. Hosts | Common Use |
|--------|---------------|------------|
| /32 | 1 | Single IPv4 host |
| /30 | 2 usable | Point-to-point links |
| /24 | 254 usable | Small subnet |
| /16 | 65,534 usable | Large private range |

CIDR is the language used by cloud platforms, route tables, firewalls, and VPNs. If engineers cannot reason about prefix size, they will struggle to design or debug network boundaries.

---

## Subnets and Gateways

A subnet groups nearby addresses into a broadcast or routing domain. Hosts in the same subnet can usually communicate directly at layer 2. Traffic for destinations outside the subnet is sent to a default gateway.

```text
Host:        10.0.1.25/24
Subnet:      10.0.1.0/24
Gateway:     10.0.1.1
Remote IP:   10.0.2.20
Action:      send traffic to gateway because destination is outside local subnet
```

Good subnet design reduces blast radius, improves policy control, and avoids accidental overlap.

---

## Private Address Space and NAT

Private IPv4 ranges:

- `10.0.0.0/8`
- `172.16.0.0/12`
- `192.168.0.0/16`

These ranges are not routable on the public internet. NAT translates private addresses to one or more public addresses for internet access.

NAT is practical, but it introduces trade-offs:

- Makes inbound connectivity more complex
- Obscures the true client address unless headers or logs preserve it
- Can create state exhaustion on busy gateways

---

## Subnet Design Guidelines

- Keep address ranges non-overlapping across VPCs, data centers, and VPN domains
- Reserve space for future growth instead of filling every range immediately
- Segment by trust boundary or workload type, not only by team name
- Document default gateways, DHCP ranges, and service IP reservations
- Avoid unnecessarily huge flat networks that are hard to secure and reason about

---

## Common Mistakes

1. **Overlapping CIDRs** — breaks peering, routing, and VPN connectivity
2. **No growth space** — forces painful renumbering later
3. **Mixing trust zones** — puts databases and user-facing services in the same range without justification
4. **Forgetting NAT effects** — application logs record gateway IPs instead of real clients

---

## Next Steps

- Continue with [03-ROUTING-AND-SWITCHING.md](03-ROUTING-AND-SWITCHING.md)
- Use [04-DNS-AND-NAME-RESOLUTION.md](04-DNS-AND-NAME-RESOLUTION.md) to understand how names map onto these addresses

---

## Version History

| Version | Date | Changes |
|--------|------|---------|
| 1.0 | 2026 | Initial guide to IP addressing and subnetting |
