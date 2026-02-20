# API Design Overview

## Table of Contents

1. [Overview](#overview)
2. [What is API Design and Why It Matters](#what-is-api-design-and-why-it-matters)
3. [API-First vs. Code-First Development](#api-first-vs-code-first-development)
4. [API Styles Overview](#api-styles-overview)
   - [REST](#rest)
   - [GraphQL](#graphql)
   - [gRPC](#grpc)
   - [WebSocket](#websocket)
   - [Webhooks](#webhooks)
5. [The API Lifecycle](#the-api-lifecycle)
6. [Key Design Principles](#key-design-principles)
   - [Consistency](#consistency)
   - [Simplicity](#simplicity)
   - [Discoverability](#discoverability)
   - [Backward Compatibility](#backward-compatibility)
7. [The Richardson Maturity Model](#the-richardson-maturity-model)
8. [API Governance and Standards](#api-governance-and-standards)
9. [Prerequisites](#prerequisites)
10. [Next Steps](#next-steps)
11. [Version History](#version-history)

---

## Overview

This documentation provides a comprehensive introduction to API design in modern software development. It covers foundational concepts, design philosophies, architectural styles, lifecycle management, and the governance practices needed to build APIs that are consistent, scalable, and pleasant to consume.

### Target Audience

- **Backend Developers** building services that expose functionality to other systems or user interfaces
- **Frontend Developers** consuming APIs and needing predictable, well-documented contracts
- **Architects** selecting API paradigms, defining standards, and governing API portfolios
- **Product Managers** evaluating API-first strategies and understanding the developer experience

### Scope

- API-first vs. code-first development philosophies
- Comparison of API architectural styles: REST, GraphQL, gRPC, WebSocket, and webhooks
- The full API lifecycle from design through deprecation
- Core design principles that apply across all API styles
- The Richardson Maturity Model for REST API evolution
- Governance frameworks and organizational standards

---

## What is API Design and Why It Matters

**API design** is the deliberate process of defining how software components communicate. A well-designed API acts as a contract between a provider and its consumers — specifying available operations, data exchange formats, and expected behavior.

APIs are no longer internal plumbing. They are **products** with their own users, documentation, and support lifecycles. Companies like Stripe, Twilio, and GitHub have demonstrated that a great API can be the primary product itself.

### Why API Design Matters

| Concern | Impact of Poor Design | Impact of Good Design |
|---|---|---|
| **Developer experience** | Frustrated consumers, high support burden, slow adoption | Intuitive interfaces, self-service onboarding, organic growth |
| **Integration cost** | Every consumer builds custom workarounds, duplicating effort | Consistent patterns reduce integration time from weeks to hours |
| **Evolvability** | Breaking changes force costly migrations, erode trust | Backward-compatible evolution preserves consumer investment |
| **Reliability** | Ambiguous contracts lead to misuse, cascading failures | Clear contracts enable proper error handling and resilience |
| **Time to market** | Teams blocked waiting for API changes or documentation | Parallel development enabled by design-first contracts |
| **Security** | Inconsistent auth, missing validation, data leakage | Uniform security patterns enforced across all endpoints |

### The Cost of Getting It Wrong

Once an API is published and consumers depend on it, changing it becomes expensive. Every breaking change requires coordinated migrations across potentially hundreds of consumers. Investing in design *before* writing code pays dividends throughout the API's lifespan.

```
  Cost of     ▲
  Change      │         ██
              │       ██
              │     ██           Fixing a design mistake in production
              │   ██             costs 10-100x more than fixing it
              │  █               during design.
              │ █
              └──────────────────────────────────────────────►
                Design   Develop   Test   Deploy   Production
```

---

## API-First vs. Code-First Development

Two dominant philosophies shape how teams build APIs. The choice between them has profound effects on team velocity, API quality, and consumer experience.

**Code-first** means you write the implementation and generate the API specification from code (e.g., annotations). The API is a byproduct of the implementation. **API-first** means you design the API contract *before* writing any implementation code. The specification drives the implementation.

### Comparison Table

| Dimension | Code-First | API-First |
|---|---|---|
| **Workflow** | Write code → generate spec → share with consumers | Design spec → review → generate code scaffolding → implement |
| **Source of truth** | The codebase | The API specification (e.g., OpenAPI YAML) |
| **Parallel development** | Blocked — consumers wait for implementation | Enabled — frontend and backend work in parallel from the spec |
| **Contract quality** | Shaped by implementation convenience | Shaped by consumer needs and usability |
| **Breaking changes** | Easy to introduce accidentally | Caught early during spec review |
| **Documentation** | Often generated and incomplete | Specification *is* the documentation |
| **Consistency** | Varies across teams and services | Enforced by shared spec standards and linting |
| **Tooling** | Framework-specific annotations (e.g., Swagger decorators) | Language-agnostic spec editors, linters (Spectral), mock servers |
| **Best for** | Prototypes, internal tools, rapid iteration | Public APIs, multi-team organizations, platform products |

### API-First Workflow

```
  ┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │  Design  │────►│   Review &   │────►│  Generate    │────►│  Implement   │
  │  the API │     │   Validate   │     │  Artifacts   │     │  & Test      │
  │  Spec    │     │   (Linting,  │     │  (Mocks,     │     │              │
  │  (OpenAPI│     │    Team      │     │   SDKs,      │     │  Server code │
  │   / Proto│     │    Review)   │     │   Stubs)     │     │  matches the │
  │   / SDL) │     │              │     │              │     │  contract    │
  └──────────┘     └──────────────┘     └──────────────┘     └──────────────┘
       │                                       │
       ▼                                       ▼
  Version-controlled spec              Frontend / mobile / partner teams
  = single source of truth             work in parallel using mocks and spec
```

---

## API Styles Overview

Each API paradigm makes different trade-offs. Understanding these trade-offs is essential for choosing the right style for each scenario.

### Comparison Table

| Dimension | REST | GraphQL | gRPC | WebSocket | Webhooks |
|---|---|---|---|---|---|
| **Protocol** | HTTP/1.1 or HTTP/2 | HTTP/1.1 or HTTP/2 | HTTP/2 | TCP (upgraded from HTTP) | HTTP (callback) |
| **Data format** | JSON, XML, or others | JSON | Protocol Buffers (binary) | Any (commonly JSON) | JSON |
| **Paradigm** | Resource-oriented | Query-oriented | Procedure-oriented | Bidirectional streaming | Event-driven push |
| **Schema** | OpenAPI (optional) | SDL (required) | Protobuf (required) | None (ad hoc) | JSON Schema / OpenAPI |
| **Discoverability** | HATEOAS links, OpenAPI | Introspection queries | Reflection, proto files | Limited | Limited |
| **Performance** | Good | Good (but N+1 risk) | Excellent (binary, multiplexed) | Excellent for real-time | N/A (async) |
| **Caching** | Native HTTP caching | Complex (POST-based) | No built-in HTTP caching | Not applicable | Not applicable |
| **Best for** | Public APIs, CRUD | Flexible client queries | Internal microservices | Real-time features | Event notifications |
| **Learning curve** | Low | Medium | Medium-High | Medium | Low |

### REST

REST is the most widely adopted API style, using HTTP methods and URIs to model resources.

```http
GET /api/v1/orders/12345 HTTP/1.1
Host: api.example.com
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "12345",
  "status": "shipped",
  "total": 49.99,
  "links": {
    "self": "/api/v1/orders/12345",
    "items": "/api/v1/orders/12345/items"
  }
}
```

**Deep dive:** [01-REST.md](01-REST.md)

### GraphQL

GraphQL lets clients request exactly the data they need in a single query, eliminating over-fetching.

```graphql
query OrderDetails {
  order(id: "12345") {
    id
    status
    total
    items {
      name
      quantity
      price
    }
    customer {
      name
      email
    }
  }
}
```

**Deep dive:** [02-GRAPHQL.md](02-GRAPHQL.md)

### gRPC

gRPC uses Protocol Buffers for strongly typed, high-performance RPC over HTTP/2 with support for streaming.

```protobuf
syntax = "proto3";

service OrderService {
  rpc GetOrder (GetOrderRequest) returns (Order);
  rpc StreamOrderUpdates (GetOrderRequest) returns (stream OrderUpdate);
}

message GetOrderRequest { string order_id = 1; }
message Order { string id = 1; string status = 2; double total = 3; }
```

**Deep dive:** [03-GRPC.md](03-GRPC.md)

### WebSocket

WebSockets provide full-duplex communication over a persistent TCP connection, ideal for real-time features like chat and live dashboards.

```javascript
const ws = new WebSocket("wss://api.example.com/orders/12345/updates");

ws.onmessage = (event) => {
  const update = JSON.parse(event.data);
  console.log(`Order ${update.orderId} status: ${update.status}`);
};
```

### Webhooks

Webhooks invert the communication direction — instead of the client polling, the server pushes events to a client-provided callback URL.

```http
POST /webhooks/orders HTTP/1.1
Host: consumer.example.com
Content-Type: application/json
X-Webhook-Signature: sha256=abc123...

{
  "event": "order.shipped",
  "timestamp": "2025-01-15T14:30:00Z",
  "data": {
    "order_id": "12345",
    "tracking_number": "1Z999AA10123456784"
  }
}
```

---

## The API Lifecycle

APIs are not static artifacts — they have a lifecycle from business need to deprecation. Each phase has distinct activities, stakeholders, and deliverables.

```
  ┌────────┐   ┌─────────┐   ┌────────┐   ┌────────┐   ┌─────────┐   ┌───────────┐
  │ DESIGN │──►│ DEVELOP │──►│  TEST  │──►│ DEPLOY │──►│ MONITOR │──►│ DEPRECATE │
  └────────┘   └─────────┘   └────────┘   └────────┘   └─────────┘   └───────────┘
  Define the   Implement     Validate     Ship to      Observe in    Sunset with
  contract     server and    contracts,   production   production,   a migration
  and review   generate      security     behind a     track SLOs    path for
  with users   client SDKs   & load test  gateway      and alert     consumers
```

| Phase | Key Activities | Artifacts |
|---|---|---|
| **Design** | Identify use cases, define resources and operations, write the spec, review with consumers | OpenAPI spec, protobuf definitions, design review notes |
| **Develop** | Implement server logic, generate SDKs and mocks, configure gateway routes | Server code, client SDKs, mock server, gateway config |
| **Test** | Contract testing, integration testing, security scanning, load testing | Test suites, security reports, performance benchmarks |
| **Deploy** | Release to staging, run canary or blue-green deployment, publish documentation | Release notes, deployment runbook, updated developer portal |
| **Monitor** | Track latency, error rates, traffic, saturation; enforce SLOs; detect anomalies | Dashboards, alerts, SLO reports, usage analytics |
| **Deprecate** | Communicate timeline, add `Sunset` headers, provide migration guide, remove after grace period | Sunset schedule, migration guide, changelog |

### Deprecation Best Practices

```http
HTTP/1.1 200 OK
Sunset: Sat, 01 Mar 2026 00:00:00 GMT
Deprecation: true
Link: </api/v2/orders>; rel="successor-version"
```

Always give consumers a clear timeline, a migration path, and programmatic signals (headers) so automated tools can detect deprecations.

---

## Key Design Principles

These principles apply regardless of whether you are building REST, GraphQL, or gRPC APIs.

### Consistency

Every endpoint in your API should feel like it was designed by the same person. Consistency reduces cognitive load for consumers and speeds up integration.

- **Naming:** Use the same conventions everywhere (`snake_case` or `camelCase` — pick one)
- **Error format:** Return the same error structure from every endpoint
- **Pagination:** Use the same pagination pattern across all list endpoints
- **Authentication:** Use the same auth mechanism for all endpoints

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request body is invalid.",
    "details": [{ "field": "email", "message": "Must be a valid email address." }],
    "request_id": "req_abc123"
  }
}
```

### Simplicity

A great API is easy to use correctly and hard to use incorrectly. Optimize for the common case and minimize the number of concepts a consumer must learn.

- Expose **resources**, not database tables
- Use **sensible defaults** so consumers do not need to specify every parameter
- Keep endpoint counts small — prefer fewer, well-designed endpoints over many narrow ones
- Return **only the data consumers need** (or let them specify what they need, as in GraphQL)

### Discoverability

A discoverable API teaches consumers how to use it without reading every page of documentation.

- **HATEOAS links** in REST responses guide consumers to related actions
- **OpenAPI specifications** enable interactive explorers (Swagger UI, Redoc)
- **GraphQL introspection** lets clients discover the schema at runtime
- **gRPC reflection** allows tools like `grpcurl` to explore service definitions
- **Consistent naming** lets consumers guess endpoint URLs and parameter names

### Backward Compatibility

Once consumers depend on your API, you must not break their integrations. Backward-compatible changes can be made safely; breaking changes require versioning.

| Change Type | Backward Compatible? | Example |
|---|---|---|
| Adding a new optional field to a response | ✅ Yes | Adding `tracking_url` to an order response |
| Adding a new optional query parameter | ✅ Yes | Adding `?include=items` to a GET endpoint |
| Adding a new endpoint | ✅ Yes | Adding `GET /api/v1/orders/{id}/timeline` |
| Removing a field from a response | ❌ No | Removing `shipping_address` from order |
| Renaming a field | ❌ No | Changing `order_total` to `total_amount` |
| Changing a field's data type | ❌ No | Changing `total` from string to number |
| Making an optional parameter required | ❌ No | Requiring `region` on all requests |
| Changing an endpoint's URL | ❌ No | Moving `/orders` to `/purchases` |

**Deep dive on versioning strategies:** [04-API-VERSIONING.md](04-API-VERSIONING.md)

---

## The Richardson Maturity Model

The Richardson Maturity Model, introduced by Leonard Richardson, describes four levels of REST API maturity.

```
  Maturity ▲
           │  ┌───────────────────────────────────────────────────────┐
  Level 3  │  │  Hypermedia Controls (HATEOAS)                       │
           │  │  Responses include links guiding clients to actions. │
           │  ├───────────────────────────────────────────────────────┤
  Level 2  │  │  HTTP Verbs                                          │
           │  │  GET, POST, PUT, DELETE with correct status codes.   │
           │  ├───────────────────────────────────────────────────────┤
  Level 1  │  │  Resources                                           │
           │  │  Individual URIs: /orders/123, /customers/456.       │
           │  ├───────────────────────────────────────────────────────┤
  Level 0  │  │  The Swamp of POX                                    │
           │  │  Single endpoint, single verb. RPC over HTTP.        │
           │  └───────────────────────────────────────────────────────┘
           └──────────────────────────────────────────────────────────►
                                                           RESTfulness
```

### Level-by-Level Breakdown

| Level | Name | Characteristics | Example |
|---|---|---|---|
| **0** | Swamp of POX | Single URI, single verb (POST). RPC-style. | `POST /api` with action in the body |
| **1** | Resources | Multiple URIs for different resources, but still using POST for everything. | `POST /orders`, `POST /customers` |
| **2** | HTTP Verbs | Correct use of HTTP methods and status codes for each operation. | `GET /orders/123` → 200, `POST /orders` → 201, `DELETE /orders/123` → 204 |
| **3** | Hypermedia | Responses contain links (HATEOAS) that tell clients what they can do next. | Response includes `"links": { "cancel": "/orders/123/cancel" }` |

Most production REST APIs operate at **Level 2**. Level 3 (HATEOAS) adds significant value for public APIs but introduces complexity that may not be justified for internal services.

**Deep dive on REST design:** [01-REST.md](01-REST.md)

---

## API Governance and Standards

As organizations grow and operate more APIs, consistency becomes a major challenge. **API governance** is the set of processes, tools, and standards that ensure all APIs meet a baseline quality bar. Without it, each team builds APIs differently and consumers must learn a new "dialect" for every service.

| Area | What to Standardize | Example Standard |
|---|---|---|
| **Naming** | URL patterns, field names, casing | `snake_case` for JSON fields, plural nouns for collections (`/orders`, not `/order`) |
| **Error handling** | Error response schema, error codes | RFC 7807 Problem Details format for all error responses |
| **Authentication** | Auth mechanism and token format | OAuth 2.0 with JWT bearer tokens; API keys for machine-to-machine |
| **Versioning** | Strategy and lifecycle policy | URI path versioning (`/v1/`); 12-month deprecation window |
| **Pagination** | Pagination pattern and parameters | Cursor-based with `page_size` and `page_token` parameters |
| **Rate limiting** | Headers and response codes | `429 Too Many Requests` with `Retry-After` and `X-RateLimit-*` headers |
| **Documentation** | Spec format and hosting | OpenAPI 3.1 specs in each repo; published to a central developer portal |

### Enforcement Tools

- **Spectral** — Lint OpenAPI specifications against custom rulesets
- **Buf** — Lint and detect breaking changes in Protocol Buffer definitions
- **CI/CD gates** — Block merges that introduce breaking API changes or fail lint checks

```yaml
# Example Spectral ruleset (.spectral.yml)
extends: "spectral:oas"
rules:
  operation-operationId:
    severity: error
  path-keys-no-trailing-slash:
    severity: warn
```

---

## Prerequisites

### Required Knowledge

Before working through the API design series, you should be familiar with:

| Topic | Why It Matters |
|---|---|
| **HTTP fundamentals** | Methods, status codes, headers, and content negotiation are the foundation of REST and webhook APIs |
| **JSON** | The dominant data format for REST, GraphQL, and webhook payloads |
| **A backend programming language** | Python, JavaScript/TypeScript, Go, or Java for implementing API examples |
| **Basic networking** | DNS, ports, TLS, and load balancers — APIs communicate over the network |
| **Git and version control** | API specs should be version-controlled alongside code |
| **Command-line tools** | `curl`, `httpie`, or `grpcurl` for testing APIs from the terminal |

### Required Tools

Install the following tools to follow hands-on examples:

```bash
# curl (usually pre-installed)
curl --version

# HTTPie — a friendlier alternative to curl
brew install httpie          # macOS
sudo apt-get install httpie  # Ubuntu/Debian

# Node.js (for JavaScript examples and tooling)
brew install node            # macOS

# Spectral — OpenAPI linter
npm install -g @stoplight/spectral-cli

# grpcurl — for testing gRPC services
brew install grpcurl         # macOS
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest  # Linux (Go)

# jq — for parsing JSON responses
brew install jq              # macOS
sudo apt-get install jq      # Ubuntu/Debian

# Buf — Protocol Buffer linter and breaking change detector
brew install bufbuild/buf/buf  # macOS
```

---

## Next Steps

Work through the API design series in order, or jump to the topic most relevant to you:

| File | Topic | Description |
|---|---|---|
| [01-REST.md](01-REST.md) | REST | RESTful conventions, HTTP methods, status codes, resource modelling, HATEOAS |
| [02-GRAPHQL.md](02-GRAPHQL.md) | GraphQL | Schema design, resolvers, subscriptions, federation, N+1 problem |
| [03-GRPC.md](03-GRPC.md) | gRPC | Protocol Buffers, streaming patterns, service definitions, deadlines |
| [04-API-VERSIONING.md](04-API-VERSIONING.md) | Versioning | Versioning strategies, backward compatibility, deprecation policies |
| [05-AUTHENTICATION-AND-AUTHORIZATION.md](05-AUTHENTICATION-AND-AUTHORIZATION.md) | Auth | OAuth 2.0, OpenID Connect, JWT, API keys, scopes |
| [06-RATE-LIMITING-AND-THROTTLING.md](06-RATE-LIMITING-AND-THROTTLING.md) | Rate Limiting | Token bucket, sliding window, quotas, backpressure |
| [07-DOCUMENTATION.md](07-DOCUMENTATION.md) | Documentation | OpenAPI/Swagger, developer portals, code generation |
| [08-TESTING.md](08-TESTING.md) | Testing | Contract testing, API mocking, integration testing, load testing |
| [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) | Best Practices | Naming conventions, pagination, error handling, idempotency |
| [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md) | Anti-Patterns | Common API design mistakes and how to avoid them |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Structured learning guide with exercises and knowledge checks |

### Suggested Learning Path by Role

```
Backend Developer:
  00-OVERVIEW → 01-REST → 04-API-VERSIONING → 05-AUTH → 09-BEST-PRACTICES

Frontend Developer:
  00-OVERVIEW → 01-REST → 02-GRAPHQL → 07-DOCUMENTATION → 08-TESTING

Architect:
  00-OVERVIEW → 01-REST → 02-GRAPHQL → 03-GRPC → 04-API-VERSIONING → 10-ANTI-PATTERNS

API Product Manager:
  00-OVERVIEW → 07-DOCUMENTATION → 04-API-VERSIONING → LEARNING-PATH
```

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial API design overview documentation |
