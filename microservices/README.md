# Microservices Architecture Learning Resources

A comprehensive guide to microservices architecture — from design principles and communication patterns to production-grade deployment, security, and observability.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | Core concepts, monolith vs. microservices, when to adopt | **Start here** |
| [01-DESIGN-PRINCIPLES](01-DESIGN-PRINCIPLES.md) | Single responsibility, loose coupling, DDD, bounded contexts | When designing services |
| [02-ARCHITECTURE-PATTERNS](02-ARCHITECTURE-PATTERNS.md) | API gateway, BFF, service mesh, sidecar, strangler fig | When choosing architecture patterns |
| [03-COMMUNICATION-PATTERNS](03-COMMUNICATION-PATTERNS.md) | REST, gRPC, async messaging, event-driven architecture | When connecting services |
| [04-DATA-MANAGEMENT](04-DATA-MANAGEMENT.md) | Database per service, saga, CQRS, event sourcing, outbox | When designing data strategies |
| [05-SERVICE-DISCOVERY](05-SERVICE-DISCOVERY.md) | Client-side, server-side, DNS-based, service registries | When configuring service discovery |
| [06-RESILIENCE-PATTERNS](06-RESILIENCE-PATTERNS.md) | Circuit breaker, retry, bulkhead, timeout, fallback | When building fault-tolerant systems |
| [07-SECURITY](07-SECURITY.md) | Zero trust, OAuth2/OIDC, mTLS, API security, secrets | When securing your services |
| [08-OBSERVABILITY](08-OBSERVABILITY.md) | Distributed tracing, centralized logging, metrics, health checks | When building observability |
| [09-DEPLOYMENT-STRATEGIES](09-DEPLOYMENT-STRATEGIES.md) | Blue-green, canary, rolling updates, feature flags, GitOps | When planning deployments |
| [10-TESTING-STRATEGIES](10-TESTING-STRATEGIES.md) | Contract testing, integration testing, chaos engineering | When designing test strategies |
| [11-BEST-PRACTICES](11-BEST-PRACTICES.md) | Production readiness, team organization, documentation | **Essential — production checklist** |
| [12-ANTI-PATTERNS](12-ANTI-PATTERNS.md) | Distributed monolith, shared databases, chatty services | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured 12-week training guide | **Start here** after the Overview |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand what microservices are
   - Compare with monolithic architecture
   - Learn when to adopt microservices

2. **Learn Design Principles** ([01-DESIGN-PRINCIPLES](01-DESIGN-PRINCIPLES.md))
   - Single responsibility and bounded contexts
   - Loose coupling and high cohesion
   - Domain-Driven Design fundamentals

3. **Understand Communication** ([03-COMMUNICATION-PATTERNS](03-COMMUNICATION-PATTERNS.md))
   - Synchronous vs. asynchronous
   - REST, gRPC, and event-driven messaging

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured, phased curriculum
   - Hands-on exercises and knowledge checks

### For Experienced Users

1. **Review Best Practices** ([11-BEST-PRACTICES](11-BEST-PRACTICES.md))
   - Production readiness checklist
   - Team organization patterns
   - Documentation standards

2. **Avoid Anti-Patterns** ([12-ANTI-PATTERNS](12-ANTI-PATTERNS.md))
   - Distributed monolith pitfalls
   - Shared database traps
   - Common operational mistakes

3. **Deep Dive into Resilience** ([06-RESILIENCE-PATTERNS](06-RESILIENCE-PATTERNS.md))
   - Circuit breaker and retry strategies
   - Bulkhead isolation
   - Graceful degradation

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        Clients                              │
│              (Web, Mobile, Third-Party)                     │
└──────────────────────┬──────────────────────────────────────┘
                       │
                ┌──────▼──────┐
                │ API Gateway │
                │ (routing,   │
                │  auth, rate │
                │  limiting)  │
                └──┬───┬───┬──┘
         ┌─────────┘   │   └──────────┐
         ▼             ▼              ▼
   ┌───────────┐ ┌───────────┐ ┌───────────┐
   │  Order    │ │  Payment  │ │  Inventory│
   │  Service  │ │  Service  │ │  Service  │
   │           │ │           │ │           │
   │  ┌─────┐  │ │  ┌─────┐  │ │  ┌─────┐  │
   │  │ DB  │  │ │  │ DB  │  │ │  │ DB  │  │
   │  └─────┘  │ │  └─────┘  │ │  └─────┘  │
   └─────┬─────┘ └─────┬─────┘ └─────┬─────┘
         │             │              │
         └──────┬──────┴──────┬───────┘
                │             │
         ┌──────▼──────┐ ┌───▼────────┐
         │ Message     │ │ Service    │
         │ Broker      │ │ Mesh       │
         │ (Kafka,     │ │ (Istio,    │
         │  RabbitMQ)  │ │  Linkerd)  │
         └─────────────┘ └────────────┘
```

## 🔑 Key Concepts

### Core Principles

```
Microservices Architecture
├── Single Responsibility      (one service, one business capability)
├── Loose Coupling             (services change independently)
├── High Cohesion              (related functionality together)
├── Database per Service       (no shared data stores)
├── Smart Endpoints            (logic in services, not pipes)
└── Decentralized Governance   (teams own their services)
```

### Communication Patterns

```
Synchronous                          Asynchronous
├── REST/HTTP                        ├── Message Queues (RabbitMQ, SQS)
├── gRPC                             ├── Event Streams (Kafka, Kinesis)
└── GraphQL                          └── Pub/Sub (SNS, Google Pub/Sub)
```

## 📋 Topics Covered

- **Fundamentals** — Core concepts, monolith vs. microservices, when to adopt
- **Design Principles** — Single responsibility, DDD, bounded contexts, loose coupling
- **Architecture** — API gateway, BFF, service mesh, sidecar, strangler fig
- **Communication** — REST, gRPC, async messaging, event-driven patterns
- **Data Management** — Database per service, saga, CQRS, event sourcing
- **Service Discovery** — Client-side, server-side, DNS-based, registries
- **Resilience** — Circuit breaker, retry, bulkhead, timeout, fallback
- **Security** — Zero trust, OAuth2/OIDC, mTLS, API security
- **Observability** — Distributed tracing, centralized logging, metrics
- **Deployment** — Blue-green, canary, rolling updates, GitOps
- **Testing** — Contract testing, integration testing, chaos engineering
- **Best Practices** — Production readiness, team organization, documentation
- **Anti-Patterns** — Distributed monolith, shared databases, chatty services

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to microservices?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md)

**Designing a new system?** → Read [01-DESIGN-PRINCIPLES.md](01-DESIGN-PRINCIPLES.md) and [02-ARCHITECTURE-PATTERNS.md](02-ARCHITECTURE-PATTERNS.md)

**Going to production?** → Review [11-BEST-PRACTICES.md](11-BEST-PRACTICES.md) and [12-ANTI-PATTERNS.md](12-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [Learning Path](LEARNING-PATH.md)
