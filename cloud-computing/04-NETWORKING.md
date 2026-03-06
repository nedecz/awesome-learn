# Cloud Networking

## Table of Contents

- [Overview](#overview)
- [Virtual Private Clouds (VPCs)](#virtual-private-clouds-vpcs)
- [Subnets](#subnets)
- [Internet and NAT Gateways](#internet-and-nat-gateways)
- [Security Groups and Firewalls](#security-groups-and-firewalls)
- [Load Balancers](#load-balancers)
- [DNS in the Cloud](#dns-in-the-cloud)
- [VPC Peering and Transit](#vpc-peering-and-transit)
- [Content Delivery Networks](#content-delivery-networks)
- [Hybrid Connectivity](#hybrid-connectivity)
- [Network Security Best Practices](#network-security-best-practices)
- [Next Steps](#next-steps)
- [Version History](#version-history)

## Overview

Cloud networking is the foundation that connects every resource, service, and user in a cloud environment. Understanding how virtual networks are designed, secured, and optimized is essential for building reliable, performant, and secure cloud architectures.

**Target Audience:** DevOps engineers, cloud architects, backend developers, and infrastructure engineers who need to design, deploy, or troubleshoot cloud networking infrastructure.

**Scope:** This document covers core networking concepts across the three major cloud providers — AWS, Azure, and Google Cloud Platform (GCP). It focuses on practical patterns, cross-provider comparisons, and security best practices for production environments.

---

## Virtual Private Clouds (VPCs)

A Virtual Private Cloud (VPC) is a logically isolated virtual network within a cloud provider's infrastructure. It gives you full control over IP address ranges, subnets, route tables, and network gateways — effectively providing your own private data center in the cloud.

### Why VPCs Matter

- **Isolation:** Resources inside a VPC are isolated from other customers and from the public internet by default.
- **Control:** You define the IP address space, create subnets, configure route tables, and manage gateways.
- **Security:** VPCs serve as the first boundary for network-level access control.

### Default vs Custom VPCs

Most cloud providers create a **default VPC** in each region when you first create an account. Default VPCs come pre-configured with public subnets, an internet gateway, and permissive routing — convenient for getting started but not suitable for production workloads.

**Custom VPCs** are created explicitly and give you full control over:

- CIDR block allocation
- Subnet placement across availability zones
- Route table configuration
- Gateway attachments

### Cross-Provider Comparison

| Feature | AWS VPC | Azure VNet | GCP VPC |
|---|---|---|---|
| Name | Virtual Private Cloud | Virtual Network (VNet) | Virtual Private Cloud |
| Scope | Regional | Regional | Global (subnets are regional) |
| Default created | Yes (per region) | No | Yes (auto-mode) |
| Max per region | 5 (adjustable) | 50 (adjustable) | 15 per project (adjustable) |
| IP range | /16 to /28 CIDR | /8 to /29 CIDR | /8 to /29 CIDR |
| IPv6 support | Yes | Yes | Yes |

### Example: Creating a VPC

```bash
# AWS CLI — Create a custom VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=production-vpc}]'

# Azure CLI — Create a VNet
az network vnet create \
  --name production-vnet \
  --resource-group my-rg \
  --address-prefix 10.0.0.0/16

# GCP — Create a custom-mode VPC
gcloud compute networks create production-vpc \
  --subnet-mode=custom
```

---

## Subnets

Subnets divide a VPC's IP address space into smaller, manageable segments. Each subnet resides in a single availability zone (or zone in GCP) and can be designated as public or private based on its routing configuration.

### Public vs Private Subnets

- **Public Subnets:** Have a route to an internet gateway. Resources with public IP addresses can communicate directly with the internet. Used for load balancers, bastion hosts, and NAT gateways.
- **Private Subnets:** Have no direct route to the internet. Resources here access the internet through a NAT gateway or NAT instance. Used for application servers, databases, and internal services.

### CIDR Block Notation

CIDR (Classless Inter-Domain Routing) notation defines the IP range of a subnet. The number after the slash indicates how many bits are fixed in the network prefix.

```
CIDR Notation Examples:
─────────────────────────────────────────────────────
10.0.0.0/8      → 16,777,216 addresses  (Class A)
10.0.0.0/16     → 65,536 addresses      (typical VPC)
10.0.0.0/24     → 256 addresses         (typical subnet)
10.0.0.0/28     → 16 addresses          (small subnet)

Common VPC Layout:
─────────────────────────────────────────────────────
VPC:              10.0.0.0/16
├── Public  AZ-a: 10.0.1.0/24   (251 usable IPs)
├── Public  AZ-b: 10.0.2.0/24   (251 usable IPs)
├── Private AZ-a: 10.0.10.0/24  (251 usable IPs)
├── Private AZ-b: 10.0.11.0/24  (251 usable IPs)
├── Data    AZ-a: 10.0.20.0/24  (251 usable IPs)
└── Data    AZ-b: 10.0.21.0/24  (251 usable IPs)
```

> **Note:** Cloud providers reserve a few IP addresses in each subnet for internal use. AWS reserves 5 IPs per subnet, Azure reserves 5, and GCP reserves 4.

### Subnet Design Best Practices

1. **Separate by function:** Use distinct subnets for public-facing, application, and data tiers.
2. **Span availability zones:** Place subnets in multiple AZs for high availability.
3. **Plan for growth:** Allocate generous CIDR blocks — renumbering later is painful.
4. **Avoid overlapping ranges:** Especially important if you plan VPC peering or hybrid connectivity.

---

## Internet and NAT Gateways

Gateways control how traffic flows between your VPC and the outside world. The two most important gateway types are the **Internet Gateway** and the **NAT Gateway**.

### Internet Gateway

An internet gateway enables resources in public subnets to send and receive traffic from the internet. It performs one-to-one NAT for instances with public IP addresses.

### NAT Gateway

A NAT (Network Address Translation) gateway allows resources in private subnets to initiate outbound connections to the internet (e.g., downloading software updates) while preventing inbound connections from the internet.

### Traffic Flow Diagram

```
                        ┌──────────────┐
                        │   Internet   │
                        └──────┬───────┘
                               │
                     ┌─────────┴─────────┐
                     │  Internet Gateway  │
                     └─────────┬─────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
      ┌───────┴──────┐  ┌─────┴───────┐  ┌─────┴───────┐
      │ Public Subnet │  │ Public Subnet│  │ NAT Gateway │
      │  (AZ-a)      │  │  (AZ-b)     │  │ (in public  │
      │  - ALB       │  │  - ALB      │  │   subnet)   │
      └──────────────┘  └─────────────┘  └──────┬──────┘
                                                 │
                                    ┌────────────┼────────────┐
                                    │                         │
                            ┌───────┴───────┐         ┌──────┴────────┐
                            │ Private Subnet│         │ Private Subnet│
                            │  (AZ-a)       │         │  (AZ-b)       │
                            │  - App servers│         │  - App servers│
                            └───────────────┘         └───────────────┘
```

**How it works:**
1. Inbound traffic from the internet hits the internet gateway and is routed to public subnets.
2. Resources in public subnets (like load balancers) forward requests to private subnets.
3. Private subnet resources that need outbound internet access send traffic through the NAT gateway, which resides in a public subnet.

### Cross-Provider Gateway Comparison

| Feature | AWS | Azure | GCP |
|---|---|---|---|
| Internet Gateway | Internet Gateway | Implicit (VNet default) | Implicit (default route) |
| NAT | NAT Gateway | NAT Gateway | Cloud NAT |
| Cost model | Hourly + data processed | Hourly + data processed | Per-VM + data processed |
| High availability | Per-AZ (deploy in each) | Zonal or regional | Regional by default |

---

## Security Groups and Firewalls

Network security in the cloud operates at multiple layers. The two primary mechanisms are **security groups** (stateful) and **network access control lists (NACLs)** (stateless).

### Security Groups (Stateful)

Security groups act as virtual firewalls at the instance/resource level. They are **stateful** — if you allow inbound traffic, the response traffic is automatically allowed regardless of outbound rules.

- Attached to individual resources (EC2 instances, RDS databases, etc.)
- Default: deny all inbound, allow all outbound
- Rules reference IP ranges or other security groups

### Network ACLs (Stateless)

NACLs operate at the subnet level and are **stateless** — you must explicitly define both inbound and outbound rules. They process rules in order by rule number.

### Comparison Table

| Feature | Security Groups | Network ACLs |
|---|---|---|
| Level | Instance / resource | Subnet |
| Statefulness | Stateful | Stateless |
| Rule type | Allow only | Allow and Deny |
| Rule evaluation | All rules evaluated | Rules processed in order |
| Default | Deny inbound, allow outbound | Allow all inbound and outbound |
| Provider support | AWS, Azure (NSG), GCP (firewall rules) | AWS (NACLs), Azure (via NSGs at subnet) |

### Example: Security Group Configuration

```yaml
# AWS Security Group rules (conceptual YAML representation)
SecurityGroup: web-server-sg
  Inbound:
    - Protocol: tcp
      Port: 443
      Source: 0.0.0.0/0          # HTTPS from anywhere
    - Protocol: tcp
      Port: 80
      Source: 0.0.0.0/0          # HTTP from anywhere (redirect to HTTPS)
    - Protocol: tcp
      Port: 22
      Source: 10.0.0.0/16        # SSH from VPC only
  Outbound:
    - Protocol: tcp
      Port: 5432
      Destination: sg-database   # PostgreSQL to database security group
    - Protocol: tcp
      Port: 443
      Destination: 0.0.0.0/0    # HTTPS outbound for API calls
```

```bash
# AWS CLI — Create a security group rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-0123456789abcdef0 \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0
```

---

## Load Balancers

Load balancers distribute incoming traffic across multiple targets to improve availability, fault tolerance, and performance. Cloud providers offer different types of load balancers optimized for different use cases.

### Layer 4 vs Layer 7

- **Layer 4 (Transport):** Routes traffic based on IP address and TCP/UDP port. Faster, lower overhead. Best for non-HTTP workloads or when you need raw performance.
- **Layer 7 (Application):** Routes traffic based on HTTP/HTTPS content — URL paths, headers, cookies. Enables advanced routing, SSL termination, and path-based routing.

### Load Balancer Types Across Clouds

| Type | AWS | Azure | GCP |
|---|---|---|---|
| Layer 7 (HTTP/S) | Application Load Balancer (ALB) | Application Gateway | HTTP(S) Load Balancer |
| Layer 4 (TCP/UDP) | Network Load Balancer (NLB) | Azure Load Balancer | TCP/UDP Load Balancer |
| Global | Global Accelerator | Front Door | Global HTTP(S) LB |
| Internal | Internal ALB / NLB | Internal LB | Internal HTTP(S) LB |

### Health Checks

Load balancers continuously monitor the health of their targets. Unhealthy targets are automatically removed from the pool until they recover.

```yaml
# Health check configuration example
HealthCheck:
  Protocol: HTTP
  Path: /health
  Port: 8080
  Interval: 30          # seconds between checks
  Timeout: 5            # seconds to wait for response
  HealthyThreshold: 3   # consecutive successes to mark healthy
  UnhealthyThreshold: 2 # consecutive failures to mark unhealthy
```

### Common Patterns

- **Path-based routing:** `/api/*` goes to API servers, `/static/*` goes to a CDN or S3 origin.
- **Host-based routing:** `api.example.com` routes to API targets, `app.example.com` routes to web targets.
- **Weighted routing:** Gradually shift traffic during blue-green or canary deployments.

---

## DNS in the Cloud

DNS (Domain Name System) translates human-readable domain names into IP addresses. Cloud-managed DNS services offer high availability, low latency, and integration with other cloud resources.

### Managed DNS Services

| Feature | AWS Route 53 | Azure DNS | GCP Cloud DNS |
|---|---|---|---|
| Global anycast | Yes | Yes | Yes |
| SLA | 100% | 100% | 100% |
| DNSSEC | Yes | Yes (public zones) | Yes |
| Private zones | Yes | Yes | Yes |
| Health checks | Yes (built-in) | Via Traffic Manager | Via health checks |
| Cost | Per hosted zone + queries | Per zone + queries | Per zone + queries |

### DNS Routing Policies

Cloud DNS services support advanced routing policies that go beyond simple name resolution:

1. **Simple Routing:** Maps a domain to a single resource. One record, one answer.
2. **Weighted Routing:** Distributes traffic across multiple resources by assigned weight. Useful for gradual rollouts.
3. **Latency-Based Routing:** Routes users to the region with the lowest network latency. Ideal for global applications.
4. **Failover Routing:** Primary/secondary configuration. Automatically fails over to a secondary resource when the primary is unhealthy.
5. **Geolocation Routing:** Routes traffic based on the geographic location of the user. Useful for compliance or localized content.

```bash
# AWS Route 53 — Create a weighted DNS record
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "blue",
        "Weight": 80,
        "TTL": 60,
        "ResourceRecords": [{"Value": "10.0.1.100"}]
      }
    }]
  }'
```

---

## VPC Peering and Transit

As cloud architectures grow, you often need to connect multiple VPCs together. VPC peering and transit gateways provide this inter-VPC connectivity.

### VPC Peering

VPC peering creates a direct, private network connection between two VPCs. Traffic flows over the cloud provider's backbone — never traversing the public internet.

**Limitations:**
- Non-transitive — if VPC-A peers with VPC-B, and VPC-B peers with VPC-C, VPC-A cannot reach VPC-C through VPC-B.
- CIDR blocks must not overlap.
- Cross-region peering is supported but may incur additional data transfer costs.

### Transit Gateway (Hub-and-Spoke)

A transit gateway acts as a central hub that connects multiple VPCs, VPN connections, and on-premises networks. It eliminates the need for complex mesh peering topologies.

### Architecture Diagram

```
     VPC Peering (Mesh)                Transit Gateway (Hub-and-Spoke)
     ──────────────────                ────────────────────────────────

    ┌─────┐     ┌─────┐              ┌─────┐   ┌─────┐   ┌─────┐
    │VPC-A├─────┤VPC-B│              │VPC-A│   │VPC-B│   │VPC-C│
    └──┬──┘     └──┬──┘              └──┬──┘   └──┬──┘   └──┬──┘
       │           │                    │         │         │
       │  ┌─────┐  │                    └─────────┼─────────┘
       └──┤VPC-C├──┘                         ┌────┴────┐
          └─────┘                            │ Transit │
                                             │ Gateway │
    3 VPCs = 3 peering connections           └────┬────┘
    N VPCs = N*(N-1)/2 connections                │
                                             ┌────┴────┐
                                             │On-Prem  │
                                             │ (VPN)   │
                                             └─────────┘
                                         All VPCs = 1 hub connection each
```

### Cross-Provider Transit Options

| Feature | AWS | Azure | GCP |
|---|---|---|---|
| Peering | VPC Peering | VNet Peering | VPC Network Peering |
| Transit hub | Transit Gateway | Virtual WAN / VNet Hub | Network Connectivity Center |
| Transitive routing | Via Transit Gateway | Via Virtual WAN | Via NCC |
| Cross-region | Yes | Yes (global peering) | Yes (global by default) |

---

## Content Delivery Networks

A Content Delivery Network (CDN) caches content at edge locations around the world, reducing latency for end users by serving content from the nearest point of presence (PoP).

### How CDNs Work

1. A user requests content (e.g., an image or API response).
2. The request is routed to the nearest CDN edge location via DNS.
3. If the content is cached (cache hit), the edge serves it directly — very fast.
4. If not cached (cache miss), the edge fetches it from the origin server, caches it, and returns it to the user.

### CDN Services Across Clouds

| Feature | AWS CloudFront | Azure CDN / Front Door | GCP Cloud CDN |
|---|---|---|---|
| Edge locations | 600+ PoPs | 190+ PoPs (Front Door) | 180+ PoPs |
| Origin types | S3, ALB, custom | Blob Storage, App Service, custom | Cloud Storage, GCE, custom |
| SSL/TLS | Free ACM certs | Free managed certs | Free managed certs |
| WebSocket support | Yes | Yes (Front Door) | Yes |
| Real-time logs | Yes | Yes | Yes |
| WAF integration | AWS WAF | Azure WAF | Cloud Armor |

### Caching Best Practices

- **Set appropriate TTLs:** Balance freshness vs performance. Static assets can have long TTLs (days/weeks); dynamic content should have short TTLs or bypass cache.
- **Use cache keys wisely:** Include only necessary query parameters in cache keys to maximize cache hit ratio.
- **Invalidation strategy:** Use versioned file names (e.g., `app.v2.js`) over cache invalidation when possible — invalidation can be slow and expensive.

```bash
# AWS CloudFront — Create a cache invalidation
aws cloudfront create-invalidation \
  --distribution-id E1234567890 \
  --paths "/images/*" "/css/*"
```

---

## Hybrid Connectivity

Hybrid connectivity bridges your on-premises data center with cloud resources. The two primary approaches are **VPN** (over the public internet) and **dedicated private connections**.

### VPN (Virtual Private Network)

VPN connections encrypt traffic between your on-premises network and your cloud VPC over the public internet. They are quick to set up and cost-effective for moderate workloads.

- **Site-to-Site VPN:** Persistent connection between your on-prem router and the cloud VPN gateway.
- **Client VPN:** Individual users connect to cloud resources from their devices.

### Dedicated Private Connections

For workloads requiring higher bandwidth, lower latency, and more consistent performance, dedicated connections bypass the public internet entirely.

| Feature | AWS Direct Connect | Azure ExpressRoute | GCP Cloud Interconnect |
|---|---|---|---|
| Bandwidth | 1 Gbps – 100 Gbps | 50 Mbps – 100 Gbps | 10 Gbps – 200 Gbps |
| Latency | Consistent, low | Consistent, low | Consistent, low |
| Encryption | Optional (MACsec) | Optional (MACsec) | Optional (HA VPN overlay) |
| Redundancy | Dual connections recommended | ExpressRoute circuits | Redundant interconnects |
| Setup time | Weeks to months | Weeks to months | Weeks to months |
| Use case | High-throughput, compliance | Enterprise hybrid | Data-intensive workloads |

### When to Use Each

- **VPN:** Getting started, low-to-moderate traffic, backup connectivity, or when you need encryption by default.
- **Dedicated Connection:** Production workloads with high bandwidth requirements, latency-sensitive applications, data migration, or regulatory compliance.
- **Both:** Use a dedicated connection as primary and VPN as failover for maximum resilience.

```bash
# AWS CLI — Create a VPN gateway
aws ec2 create-vpn-gateway --type ipsec.1

# Azure CLI — Create a VPN gateway
az network vnet-gateway create \
  --name my-vpn-gateway \
  --resource-group my-rg \
  --vnet my-vnet \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw1
```

---

## Network Security Best Practices

Securing cloud networks requires a defense-in-depth strategy — multiple overlapping security controls that protect against different attack vectors at different layers.

### Defense in Depth

Apply security at every layer of the network stack:

1. **Edge:** WAF, DDoS protection, CDN-level filtering.
2. **VPC boundary:** NACLs, flow logs, traffic mirroring.
3. **Subnet:** Route table controls, network segmentation.
4. **Instance:** Security groups, host-based firewalls.
5. **Application:** TLS encryption, authentication, input validation.

### Micro-Segmentation

Instead of relying on a single perimeter, micro-segmentation applies fine-grained security policies between individual workloads.

- Use security groups to restrict traffic between tiers (web → app → database).
- Deny all traffic by default; explicitly allow only required flows.
- Regularly audit security group rules for overly permissive access.

### Private Endpoints

Private endpoints allow you to access cloud services (like S3, Azure SQL, or Cloud Storage) over private IP addresses within your VPC, eliminating exposure to the public internet.

| Feature | AWS | Azure | GCP |
|---|---|---|---|
| Service | VPC Endpoints (Gateway/Interface) | Private Endpoints | Private Service Connect |
| Supported services | S3, DynamoDB, 100+ services | SQL, Storage, 100+ services | Cloud Storage, BigQuery, 60+ services |
| DNS integration | Private hosted zones | Private DNS zones | Automatic DNS |

### Security Checklist

```
✅  Enable VPC flow logs for all subnets
✅  Use private subnets for databases and application servers
✅  Restrict security group rules to specific ports and sources
✅  Deploy NAT gateways instead of assigning public IPs to instances
✅  Use private endpoints for cloud service access
✅  Enable DDoS protection (AWS Shield, Azure DDoS Protection, Cloud Armor)
✅  Encrypt all traffic in transit with TLS
✅  Regularly review and remove unused security group rules
✅  Implement network segmentation between environments (dev, staging, prod)
✅  Use infrastructure as code to manage network configurations
```

---

## Next Steps

| Topic | Resource | Why |
|---|---|---|
| Infrastructure as Code | [Infrastructure as Code Guide](../infrastructure-as-code/) | Automate network provisioning with Terraform or CloudFormation |
| Kubernetes Networking | [Kubernetes Guide](../kubernetes/) | Understand pod networking, services, and ingress controllers |
| Security | [Security Guide](../security/) | Deep dive into cloud security beyond networking |
| Microservices | [Microservices Guide](../microservices/) | Service mesh networking and service discovery patterns |
| Observability | [Observability Guide](../observability/) | Monitor network performance, flow logs, and latency |
| System Design | [System Design Guide](../system-design/) | Design highly available, globally distributed architectures |

## Version History

| Version | Date | Description |
|---|---|---|
| 1.0 | 2025 | Initial cloud networking documentation |
