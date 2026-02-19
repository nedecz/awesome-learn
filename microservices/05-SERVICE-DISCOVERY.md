# Microservices Service Discovery

## Table of Contents

1. [Overview](#overview)
2. [Why Service Discovery Matters](#why-service-discovery-matters)
3. [Client-Side Discovery](#client-side-discovery)
4. [Server-Side Discovery](#server-side-discovery)
5. [DNS-Based Discovery](#dns-based-discovery)
6. [Service Registry](#service-registry)
7. [Health Checking](#health-checking)
8. [Service Discovery in Kubernetes](#service-discovery-in-kubernetes)
9. [Best Practices](#best-practices)
10. [Next Steps](#next-steps)

## Overview

Service discovery is the mechanism by which microservices locate and communicate with each other in a dynamic environment where service instances are constantly created, destroyed, and moved across hosts.

### Target Audience

- Platform engineers building service infrastructure
- Developers configuring inter-service communication
- DevOps engineers managing service registration and health checks

### Scope

- Client-side and server-side discovery patterns
- DNS-based discovery mechanisms
- Service registries (Consul, Eureka, etcd)
- Health checking and readiness
- Kubernetes-native service discovery

## Why Service Discovery Matters

In a microservices environment, service instances are ephemeral. They scale up and down, restart after failures, and are deployed across multiple hosts. Hardcoding IP addresses and ports is not viable.

```
Static Configuration (fragile)        Dynamic Discovery (resilient)

  Order Service                         Order Service
  config:                               config:
    payment_url: 10.0.1.5:8080           payment_service: "payment-svc"
                                         ↓
  What if 10.0.1.5 goes down?           Service Registry resolves
  What if Payment scales to             to healthy instances:
  3 instances?                            10.0.1.5:8080 ✅
                                          10.0.1.6:8080 ✅
                                          10.0.1.7:8080 ✅
```

## Client-Side Discovery

The client queries the service registry directly and selects an instance to call. The client is responsible for load balancing across available instances.

```
┌──────────────┐     1. Query registry    ┌──────────────────┐
│ Order        │ ────────────────────────▶ │ Service Registry │
│ Service      │ ◀──────────────────────── │ (Consul, Eureka) │
│              │     2. Returns instances  └──────────────────┘
│ ┌──────────┐ │           │
│ │Client-   │ │           │ payment-svc:
│ │side LB   │ │           │   10.0.1.5:8080
│ └────┬─────┘ │           │   10.0.1.6:8080
└──────┼───────┘           │   10.0.1.7:8080
       │
       │ 3. Call selected instance
       ▼
┌──────────────┐
│ Payment Svc  │
│ 10.0.1.6:8080│
└──────────────┘
```

### Pros and Cons

| Pros | Cons |
|---|---|
| Fewer network hops | Discovery logic coupled to every service |
| Client controls load balancing strategy | Every language/framework needs a client library |
| Can implement smart routing (latency-based, etc.) | Service registry becomes a critical dependency |

## Server-Side Discovery

The client sends requests to a load balancer or router, which queries the service registry and forwards the request to a healthy instance.

```
┌──────────────┐     1. Request           ┌──────────────┐
│ Order        │ ────────────────────────▶ │ Load Balancer│
│ Service      │                           │ / Router     │
└──────────────┘                           └──────┬───────┘
                                                  │
                              2. Query registry   │
                         ┌────────────────────────┤
                         ▼                        │
                  ┌──────────────────┐            │
                  │ Service Registry │            │
                  └──────────────────┘            │
                                                  │ 3. Forward to
                                                  │    healthy instance
                                                  ▼
                                           ┌──────────────┐
                                           │ Payment Svc  │
                                           │ 10.0.1.6:8080│
                                           └──────────────┘
```

### Pros and Cons

| Pros | Cons |
|---|---|
| Clients are simple — no discovery logic | Extra network hop through load balancer |
| Language-agnostic — works with any client | Load balancer is a potential bottleneck |
| Centralized load balancing and routing | More infrastructure to manage |

## DNS-Based Discovery

DNS-based discovery uses standard DNS records to resolve service names to IP addresses. This is the simplest approach and is native to most platforms.

### How It Works

```
┌──────────────┐    DNS Query:               ┌───────────┐
│ Order        │    payment-svc.prod.local    │ DNS Server│
│ Service      │ ───────────────────────────▶ │ (CoreDNS, │
│              │ ◀─────────────────────────── │  Route 53)│
└──────────────┘    A Records:                └───────────┘
                      10.0.1.5
                      10.0.1.6
                      10.0.1.7
```

### DNS Record Types for Service Discovery

| Record | Purpose | Example |
|---|---|---|
| **A record** | Maps service name to IP addresses | `payment-svc → 10.0.1.5, 10.0.1.6` |
| **SRV record** | Maps service name to IP + port | `_http._tcp.payment-svc → 10.0.1.5:8080` |
| **CNAME record** | Alias to another domain name | `payment-svc → payment.internal.example.com` |

### Limitations of DNS

- **TTL caching** — DNS responses are cached, so changes take time to propagate
- **No health checking** — DNS does not remove unhealthy instances automatically
- **Round-robin only** — DNS cannot do sophisticated load balancing

## Service Registry

A service registry is a database of available service instances, their network locations, and their health status.

### How Registration Works

```
1. Service starts            2. Register with registry    3. Heartbeat

┌──────────────┐            ┌──────────────────┐         ┌──────────────────┐
│ Payment Svc  │───────────▶│ Service Registry │         │ Service Registry │
│ (starting)   │  Register: │                  │         │                  │
└──────────────┘  name: pay │  payment-svc:    │         │  payment-svc:    │
                  host: 10. │   10.0.1.5 ✅    │◀─ beat ─│   10.0.1.5 ✅    │
                  port: 8080│                  │         │                  │
                            └──────────────────┘         └──────────────────┘

4. Deregister on shutdown (or timeout if crashed)
```

### Registry Comparison

| Registry | Protocol | Health Check | Key Features |
|---|---|---|---|
| **Consul** | HTTP, DNS, gRPC | TCP, HTTP, gRPC, script | Service mesh, K/V store, multi-DC |
| **Eureka** | HTTP | Client heartbeat | Netflix OSS, Java-centric, self-preservation |
| **etcd** | gRPC | TTL-based leases | Kubernetes backing store, strong consistency |
| **ZooKeeper** | Custom TCP | Ephemeral nodes | Mature, complex, used by Kafka |

### Self-Registration vs. Third-Party Registration

| Approach | Description | Trade-Off |
|---|---|---|
| **Self-registration** | Each service registers itself on startup | Simple; service must know about the registry |
| **Third-party registration** | A registrar (e.g., Kubernetes, Consul agent) registers services | Services are registry-agnostic; extra component needed |

## Health Checking

Service registries need to know whether instances are healthy. Unhealthy instances should be removed from the registry to prevent routing traffic to them.

### Health Check Types

| Type | Mechanism | Best For |
|---|---|---|
| **HTTP health check** | `GET /health` returns 200 OK | Web services, REST APIs |
| **TCP health check** | Open a TCP connection | Database proxies, non-HTTP services |
| **gRPC health check** | gRPC Health Checking Protocol | gRPC services |
| **Heartbeat** | Service sends periodic heartbeats | Self-registration patterns |

### Health Endpoint Design

```json
GET /health

{
  "status": "UP",
  "checks": {
    "database": { "status": "UP", "responseTime": "12ms" },
    "cache": { "status": "UP", "responseTime": "2ms" },
    "downstream-api": { "status": "UP", "responseTime": "45ms" }
  },
  "version": "2.3.1",
  "uptime": "4h 23m"
}
```

### Liveness vs. Readiness

| Check | Purpose | Failure Action |
|---|---|---|
| **Liveness** | Is the service alive? | Restart the instance |
| **Readiness** | Is the service ready to accept traffic? | Remove from load balancer (but do not restart) |

## Service Discovery in Kubernetes

Kubernetes provides built-in service discovery through Services and DNS. This is the most common approach in containerized environments.

### How It Works

```
┌──────────────┐    DNS Query:                    ┌───────────┐
│ Order Pod    │    payment-svc.prod.svc.          │  CoreDNS  │
│              │    cluster.local                  │           │
│              │ ─────────────────────────────────▶│  Watches  │
│              │ ◀─────────────────────────────────│  K8s API  │
└──────────────┘    ClusterIP: 10.96.0.50          └───────────┘
                           │
                    ┌──────▼───────┐
                    │  kube-proxy  │
                    │  (iptables   │
                    │   or IPVS)   │
                    └──────┬───────┘
                    ┌──────┼──────┐
                    ▼      ▼      ▼
              ┌──────┐ ┌──────┐ ┌──────┐
              │Pay-1 │ │Pay-2 │ │Pay-3 │
              └──────┘ └──────┘ └──────┘
```

### Kubernetes DNS Naming

| Format | Example | Scope |
|---|---|---|
| `<svc>` | `payment-svc` | Same namespace |
| `<svc>.<ns>` | `payment-svc.prod` | Cross-namespace |
| `<svc>.<ns>.svc.cluster.local` | `payment-svc.prod.svc.cluster.local` | Fully qualified |

## Best Practices

- ✅ Use the platform's native service discovery (Kubernetes Services, ECS Service Discovery)
- ✅ Implement both liveness and readiness health checks
- ✅ Use DNS-based discovery as the default; add a service registry for advanced needs
- ✅ Design services to handle discovery failures gracefully (retry, circuit break)
- ❌ Do not hardcode IP addresses or ports in service configuration
- ❌ Do not rely solely on DNS TTL for instance freshness — combine with health checks
- ❌ Do not expose internal service discovery mechanisms to external clients

## Next Steps

Continue to [Resilience Patterns](06-RESILIENCE-PATTERNS.md) to learn about circuit breakers, retries, bulkheads, timeouts, and other patterns for building fault-tolerant microservices.
