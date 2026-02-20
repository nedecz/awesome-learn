# API Design Learning Path

A structured, self-paced training guide to mastering API design — from REST fundamentals and GraphQL to production-grade versioning, security, and testing. Each phase builds on the previous one.

> **Time Estimate:** 8–10 weeks at ~5 hours/week. Adjust pace to your experience level.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds on prior knowledge
2. **Read the linked documents** — they contain detailed content, code examples, and reference tables
3. **Complete the exercises** — hands-on practice solidifies understanding
4. **Check yourself** — use the knowledge check items before moving to the next phase

---

## Phase 1: Foundations (Week 1–2)

### Learning Objectives

- Understand API design principles and the API-first approach
- Learn the differences between REST, GraphQL, and gRPC
- Grasp resource modeling and URI design fundamentals

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | API-first development, API styles comparison, API lifecycle |
| 2 | [01-REST](01-REST.md) | REST constraints, HTTP methods, status codes, resource naming |

### Exercises

1. **Design a resource model** for a library management system (books, authors, members, loans). Define URIs, HTTP methods, and status codes for each resource.
2. **Map HTTP methods** to CRUD operations for a blog API (posts, comments, tags).
3. **Identify REST constraint violations** in a provided API design.

### Knowledge Check

- [ ] Can you explain the six REST constraints?
- [ ] Can you choose the correct HTTP method and status code for a given operation?
- [ ] Can you design a URI hierarchy for a domain model?

---

## Phase 2: API Paradigms (Week 3–4)

### Learning Objectives

- Design GraphQL schemas with queries, mutations, and subscriptions
- Define gRPC services with Protocol Buffers
- Choose the right API style for a given use case

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-GRAPHQL](02-GRAPHQL.md) | Schema design, resolvers, N+1 problem, federation |
| 2 | [03-GRPC](03-GRPC.md) | Protobuf, streaming patterns, service definitions |

### Exercises

1. **Design a GraphQL schema** for an e-commerce product catalog with products, categories, and reviews.
2. **Define a gRPC service** for a notification system with unary and server-streaming RPCs.
3. **Compare approaches:** Given a scenario (mobile app, internal microservices, public API), justify which API style you would choose.

### Knowledge Check

- [ ] Can you write a GraphQL schema with types, queries, and mutations?
- [ ] Can you define a Protocol Buffers service with message types?
- [ ] Can you explain when to choose REST vs GraphQL vs gRPC?

---

## Phase 3: Evolution and Security (Week 5–6)

### Learning Objectives

- Apply API versioning strategies
- Implement authentication and authorization patterns
- Design rate limiting and throttling

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [04-API-VERSIONING](04-API-VERSIONING.md) | Versioning strategies, breaking changes, deprecation |
| 2 | [05-AUTHENTICATION-AND-AUTHORIZATION](05-AUTHENTICATION-AND-AUTHORIZATION.md) | OAuth 2.0, JWT, API keys, scopes |
| 3 | [06-RATE-LIMITING-AND-THROTTLING](06-RATE-LIMITING-AND-THROTTLING.md) | Algorithms, HTTP headers, backpressure |

### Exercises

1. **Plan a versioning strategy** for an API that needs to rename a field and remove a deprecated endpoint.
2. **Design an OAuth 2.0 flow** for a mobile app accessing a REST API.
3. **Implement a token bucket** rate limiter in your language of choice.

### Knowledge Check

- [ ] Can you identify what constitutes a breaking change?
- [ ] Can you explain the OAuth 2.0 authorization code flow with PKCE?
- [ ] Can you choose a rate limiting algorithm and explain the trade-offs?

---

## Phase 4: Documentation and Testing (Week 7–8)

### Learning Objectives

- Write OpenAPI specifications
- Set up contract testing and API mocking
- Build a CI pipeline for API validation

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [07-DOCUMENTATION](07-DOCUMENTATION.md) | OpenAPI spec, documentation tools, developer portals |
| 2 | [08-TESTING](08-TESTING.md) | Contract testing, mocking, performance testing, security testing |

### Exercises

1. **Write an OpenAPI spec** for a user management API (CRUD on users with authentication).
2. **Set up a mock server** using Prism and verify it serves example responses.
3. **Write contract tests** that validate API responses against the OpenAPI schema.
4. **Run a basic load test** with k6 against a sample API endpoint.

### Knowledge Check

- [ ] Can you write a complete OpenAPI 3.x specification?
- [ ] Can you set up contract testing in a CI pipeline?
- [ ] Can you design a testing strategy for a new API?

---

## Phase 5: Mastery (Week 9–10)

### Learning Objectives

- Apply comprehensive best practices to API design
- Identify and avoid common anti-patterns
- Conduct API design reviews

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [09-BEST-PRACTICES](09-BEST-PRACTICES.md) | Naming, pagination, errors, caching, idempotency |
| 2 | [10-ANTI-PATTERNS](10-ANTI-PATTERNS.md) | 12 common mistakes and their fixes |

### Exercises

1. **Audit an existing API** against the best practices checklist. Document findings and propose fixes.
2. **Design review:** Review a peer's API design and identify anti-patterns.
3. **Capstone project:** Design a complete API for a task management system:
   - REST API with OpenAPI spec
   - Authentication with OAuth 2.0
   - Pagination, filtering, and sorting
   - Error handling with RFC 7807
   - Versioning strategy
   - Rate limiting design
   - Documentation with developer portal

### Knowledge Check

- [ ] Can you review an API design and identify anti-patterns?
- [ ] Can you design a production-ready API from scratch?
- [ ] Can you justify every design decision with a best practice reference?

---

## Suggested Learning Path by Role

```
Backend Developer:
  00-OVERVIEW → 01-REST → 04-VERSIONING → 09-BEST-PRACTICES → 10-ANTI-PATTERNS

Frontend Developer:
  00-OVERVIEW → 01-REST → 02-GRAPHQL → 07-DOCUMENTATION → 08-TESTING

Platform / API Engineer:
  00-OVERVIEW → 01-REST → 03-GRPC → 05-AUTH → 06-RATE-LIMITING → 07-DOCUMENTATION

Architect:
  00-OVERVIEW → All files → 09-BEST-PRACTICES → 10-ANTI-PATTERNS
```

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial API design learning path |
