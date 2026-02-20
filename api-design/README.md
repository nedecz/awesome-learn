# API Design Learning Resources

A comprehensive guide to designing, building, and maintaining robust APIs — from REST and GraphQL fundamentals to gRPC, versioning strategies, authentication, rate limiting, documentation, testing, and production best practices.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | API design principles, API-first development, design lifecycle | **Start here** |
| [01-REST](01-REST.md) | RESTful conventions, HTTP methods, status codes, HATEOAS | When designing or consuming REST APIs |
| [02-GRAPHQL](02-GRAPHQL.md) | Schema design, resolvers, subscriptions, federation | When evaluating or building GraphQL APIs |
| [03-GRPC](03-GRPC.md) | Protocol Buffers, streaming patterns, service definitions | When building high-performance internal APIs |
| [04-API-VERSIONING](04-API-VERSIONING.md) | Versioning strategies, backward compatibility, deprecation | When evolving APIs without breaking clients |
| [05-AUTHENTICATION-AND-AUTHORIZATION](05-AUTHENTICATION-AND-AUTHORIZATION.md) | OAuth 2.0, OpenID Connect, API keys, JWT | When securing APIs and managing access control |
| [06-RATE-LIMITING-AND-THROTTLING](06-RATE-LIMITING-AND-THROTTLING.md) | Rate limiting patterns, quotas, backpressure | When protecting APIs from abuse and overload |
| [07-DOCUMENTATION](07-DOCUMENTATION.md) | OpenAPI/Swagger, API documentation tools, developer portals | When documenting APIs for consumers |
| [08-TESTING](08-TESTING.md) | Contract testing, API mocking, integration testing | When building API test strategies |
| [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Naming conventions, pagination, error handling, idempotency | **Essential — production checklist** |
| [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | Common API design mistakes and how to avoid them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured learning guide with exercises and knowledge checks | **Start here** after the Overview |
| [README](README.md) | This index file | Start here for navigation |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand API design principles and the API-first approach
   - Learn the API design lifecycle from concept to deprecation
   - Survey the API ecosystem: REST, GraphQL, gRPC, and beyond

2. **Learn RESTful API Design** ([01-REST](01-REST.md))
   - Master HTTP methods, status codes, and resource modelling
   - Apply HATEOAS and content negotiation
   - Design your first RESTful endpoint hierarchy

3. **Secure Your API** ([05-AUTHENTICATION-AND-AUTHORIZATION](05-AUTHENTICATION-AND-AUTHORIZATION.md))
   - Implement OAuth 2.0 authorization flows
   - Add JWT-based authentication to your endpoints
   - Understand scopes, roles, and API key management

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured multi-phase curriculum with hands-on exercises
   - Knowledge checks, design reviews, and a capstone project

### For Experienced Users

1. **Review Best Practices** ([09-BEST-PRACTICES](09-BEST-PRACTICES.md))
   - Naming conventions, pagination strategies, and error handling
   - Idempotency, caching, and performance optimization
   - Consistency patterns across large API surfaces

2. **Audit Against Anti-Patterns** ([10-ANTI-PATTERNS](10-ANTI-PATTERNS.md))
   - Common API design mistakes with real-world examples
   - Quick reference checklist for API design reviews

3. **Explore Advanced Paradigms** ([02-GRAPHQL](02-GRAPHQL.md) / [03-GRPC](03-GRPC.md))
   - Evaluate GraphQL federation for composable API gateways
   - Adopt gRPC and Protocol Buffers for internal service communication
   - Compare trade-offs across REST, GraphQL, and gRPC

4. **Plan Versioning and Evolution** ([04-API-VERSIONING](04-API-VERSIONING.md))
   - Choose between URI, header, and query parameter versioning
   - Implement backward-compatible changes and deprecation policies
   - Design sunset strategies and migration guides for consumers

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                       API Consumers                              │
│   Web Apps │ Mobile Apps │ Partner Services │ Third-Party Devs   │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTP / gRPC / WebSocket
┌────────────────────────▼────────────────────────────────────────┐
│                    API Gateway Layer                              │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │   Authentication │ Rate Limiting │ Routing │ Analytics    │  │
│   │   (OAuth 2.0, JWT, API keys, quotas, throttling)         │  │
│   └──────────────────────┬───────────────────────────────────┘  │
└──────────────────────────│──────────────────────────────────────┘
                           │ validated requests
┌──────────────────────────▼──────────────────────────────────────┐
│                    API Interface Layer                            │
│   ┌─────────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│   │   REST / HTTP   │  │   GraphQL    │  │   gRPC / Protobuf │  │
│   │   (JSON, XML)   │  │   (Schema,   │  │   (Binary, HTTP/2 │  │
│   │   HATEOAS,      │  │    Resolvers,│  │    Streaming,     │  │
│   │   Content-Type  │  │    Federation│  │    Bidirectional) │  │
│   └────────┬────────┘  └──────┬───────┘  └─────────┬─────────┘  │
└────────────│──────────────────│──────────────────────│───────────┘
             │                  │                       │
┌────────────▼──────────────────▼───────────────────────▼──────────┐
│                   Business Logic Layer                             │
│   ┌──────────────┐  ┌────────────────────┐  ┌────────────────┐   │
│   │  Validation  │  │  Domain Services   │  │  Authorization │   │
│   │  & Mapping   │  │  & Orchestration   │  │  & Policy      │   │
│   └──────────────┘  └────────────────────┘  └────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
             │                  │                       │
┌────────────▼──────────────────▼───────────────────────▼──────────┐
│                      Data / Storage Layer                          │
│   ┌──────────────┐  ┌────────────────────┐  ┌────────────────┐   │
│   │  Relational  │  │  Document / NoSQL  │  │  Cache Layer   │   │
│   │  (PostgreSQL,│  │  (MongoDB, Dynamo, │  │  (Redis,       │   │
│   │   MySQL)     │  │   Elasticsearch)   │  │   Memcached)   │   │
│   └──────────────┘  └────────────────────┘  └────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
             │
┌────────────▼──────────────────────────────────────────────────────┐
│                  Documentation & Developer Portal                  │
│   ┌─────────────────┐  ┌──────────────┐  ┌────────────────────┐   │
│   │  OpenAPI /      │  │  Interactive │  │  SDKs / Client     │   │
│   │  Swagger Spec   │──▶  API Explorer │  │  Libraries /       │   │
│   │  (YAML / JSON)  │  │  (Try It Out)│  │  Code Generation   │   │
│   └─────────────────┘  └──────────────┘  └────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
```

## 🔑 Key Concepts

```
API Design Paradigms
────────────────────
REST     → Resource-oriented architecture over HTTP (JSON, HATEOAS, stateless)
GraphQL  → Query language with client-defined response shapes (schema, resolvers)
gRPC     → High-performance RPC with Protocol Buffers (binary, HTTP/2, streaming)

HTTP Methods (REST)
───────────────────
GET      → Retrieve a resource (safe, idempotent, cacheable)
POST     → Create a resource or trigger an action (not idempotent)
PUT      → Replace a resource entirely (idempotent)
PATCH    → Partially update a resource (not necessarily idempotent)
DELETE   → Remove a resource (idempotent)

API Security
────────────
OAuth 2.0    → Delegated authorization framework (authorization code, client credentials)
OpenID Connect → Identity layer on top of OAuth 2.0 (ID tokens, user info)
JWT          → JSON Web Tokens for stateless authentication (signed claims)
API Keys     → Simple token-based access for server-to-server communication

Versioning Strategies
─────────────────────
URI Path     → /v1/users — explicit, easy to route, breaks URI stability
Header       → Accept: application/vnd.api+json;version=2 — clean URIs, harder to test
Query Param  → /users?version=2 — simple but easily overlooked

Quality Attributes
──────────────────
Consistency  → Uniform naming, error formats, and pagination across all endpoints
Discoverability → HATEOAS links, OpenAPI specs, and developer portal documentation
Resilience   → Rate limiting, circuit breakers, retries with exponential backoff
Evolvability → Backward-compatible changes, sunset headers, deprecation policies
```

## 📋 Topics Covered

- **Foundations** — API design principles, API-first development, design lifecycle, style comparison
- **REST** — HTTP methods, status codes, resource modelling, HATEOAS, content negotiation
- **GraphQL** — Schema definition language, resolvers, subscriptions, federation, N+1 problem
- **gRPC** — Protocol Buffers, unary and streaming RPCs, service definitions, deadlines
- **Versioning** — URI, header, and query parameter strategies; backward compatibility; deprecation
- **Authentication & Authorization** — OAuth 2.0 flows, OpenID Connect, JWT, API keys, scopes
- **Rate Limiting & Throttling** — Token bucket, sliding window, quotas, backpressure, retry headers
- **Documentation** — OpenAPI/Swagger, API documentation tools, developer portals, code generation
- **Testing** — Contract testing (Pact), API mocking, integration testing, load testing
- **Best Practices** — Naming conventions, pagination, error handling, idempotency, caching
- **Anti-Patterns** — Common API design mistakes with examples, fixes, and prevention strategies

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to API design?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md) then follow [LEARNING-PATH.md](LEARNING-PATH.md)

**Building a REST API?** → Jump to [01-REST.md](01-REST.md) and [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md)

**Evaluating GraphQL or gRPC?** → Compare [02-GRAPHQL.md](02-GRAPHQL.md) and [03-GRPC.md](03-GRPC.md) with REST

**Going to production?** → Review [05-AUTHENTICATION-AND-AUTHORIZATION.md](05-AUTHENTICATION-AND-AUTHORIZATION.md), [06-RATE-LIMITING-AND-THROTTLING.md](06-RATE-LIMITING-AND-THROTTLING.md), and [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [LEARNING-PATH.md](LEARNING-PATH.md) — phased curriculum with exercises
