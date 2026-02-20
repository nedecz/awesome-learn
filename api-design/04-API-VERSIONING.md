# API Versioning

## Table of Contents

1. [Overview](#overview)
2. [Why Versioning Matters](#why-versioning-matters)
3. [URI Path Versioning](#uri-path-versioning)
4. [Query Parameter Versioning](#query-parameter-versioning)
5. [Header Versioning](#header-versioning)
6. [Comparison of Versioning Strategies](#comparison-of-versioning-strategies)
7. [Semantic Versioning for APIs](#semantic-versioning-for-apis)
8. [Identifying Breaking Changes](#identifying-breaking-changes)
9. [Deprecation Strategies](#deprecation-strategies)
10. [Backward Compatibility Patterns](#backward-compatibility-patterns)
11. [API Evolution Without Versioning](#api-evolution-without-versioning)
12. [Migration Strategies](#migration-strategies)
13. [Best Practices](#best-practices)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to API versioning — the strategies, trade-offs, and patterns that allow APIs to evolve over time without breaking existing consumers. Versioning is one of the most consequential design decisions you will make for a public or internal API.

### Target Audience

- **Backend Developers** building and maintaining APIs consumed by multiple clients
- **Frontend Developers** who need stable contracts while APIs evolve
- **Architects** defining API governance and lifecycle policies
- **Tech Leads** establishing conventions for versioning across services

### Scope

- Breaking vs non-breaking change classification
- URI path, query parameter, and header-based versioning approaches
- Semantic versioning applied to HTTP APIs
- Deprecation timelines, sunset headers, and consumer communication
- Backward compatibility patterns and API evolution without explicit versions

---

## Why Versioning Matters

APIs are contracts. Once a consumer integrates with your API, any change you make has the potential to break their application. Versioning gives you a mechanism to introduce changes — including breaking ones — while giving consumers time and a clear path to migrate.

### Breaking vs Non-Breaking Changes

```
Non-Breaking (Safe)                    Breaking (Dangerous)
─────────────────────                  ─────────────────────
✅ Adding a new endpoint               ❌ Removing an endpoint
✅ Adding an optional field             ❌ Removing or renaming a field
✅ Adding a new query parameter         ❌ Changing a field's data type
✅ Widening an accepted value range     ❌ Narrowing an accepted value range
✅ Adding a new HTTP header             ❌ Making an optional field required
✅ Increasing a rate limit              ❌ Changing error response format
```

Without a versioning strategy, you face an impossible choice: never improve your API, or break every consumer with each change.

---

## URI Path Versioning

URI path versioning embeds the version number directly in the URL path. This is the most widely adopted approach, used by companies like Google, Stripe, and Twitter.

```http
GET /v1/users/42 HTTP/1.1
Host: api.example.com
Accept: application/json
```

```http
GET /v2/users/42 HTTP/1.1
Host: api.example.com
Accept: application/json
```

### Response Differences Across Versions

```json
// v1 response — name is a single string
{
  "id": 42,
  "name": "Alice Johnson",
  "email": "alice@example.com"
}

// v2 response — name split into components
{
  "id": 42,
  "first_name": "Alice",
  "last_name": "Johnson",
  "email": "alice@example.com",
  "created_at": "2025-01-15T09:30:00Z"
}
```

| Pros | Cons |
|---|---|
| Highly visible and explicit | Couples version to resource identity — the URI changes |
| Easy to route at the load balancer or gateway | Can lead to URI proliferation across many versions |
| Simple to test with curl or a browser | Violates the REST principle that a URI identifies a resource |
| Excellent tooling support (OpenAPI, docs) | Clients must update base URLs when migrating |

---

## Query Parameter Versioning

Query parameter versioning passes the version as a request parameter, keeping the resource path stable.

```http
GET /users/42?version=2 HTTP/1.1
Host: api.example.com
Accept: application/json
```

When no version parameter is provided, the API returns a well-defined default — typically the latest stable version or the oldest supported version, depending on your policy.

| Pros | Cons |
|---|---|
| Resource URI stays the same across versions | Easy to forget the parameter — default behavior must be clear |
| Optional parameter allows graceful defaults | Version can be lost when URLs are copied or shared |
| Easy to implement in application code | Query parameters are less visible than path segments |
| Clients can upgrade one endpoint at a time | Caching is more complex — vary on query string |

---

## Header Versioning

Header versioning uses HTTP headers to specify the desired API version, keeping the URI completely clean. There are two common approaches: custom headers and media type versioning.

### Custom Header

```http
GET /users/42 HTTP/1.1
Host: api.example.com
X-API-Version: 2
Accept: application/json
```

### Media Type Versioning (Accept Header)

This approach uses content negotiation via the `Accept` header, treating each version as a distinct media type. GitHub's API popularized this pattern.

```http
GET /users/42 HTTP/1.1
Host: api.example.com
Accept: application/vnd.example.v2+json
```

```http
HTTP/1.1 200 OK
Content-Type: application/vnd.example.v2+json

{
  "id": 42,
  "first_name": "Alice",
  "last_name": "Johnson",
  "email": "alice@example.com"
}
```

| Pros | Cons |
|---|---|
| URIs remain clean and version-agnostic | Harder to test — cannot simply change a URL in a browser |
| Aligns with HTTP content negotiation standards | Less visible to developers unfamiliar with headers |
| Supports fine-grained versioning per resource type | Custom headers may be stripped by proxies |
| No impact on caching by URI | Documentation and discovery are more complex |

---

## Comparison of Versioning Strategies

| Criteria | URI Path | Query Parameter | Custom Header | Media Type |
|---|---|---|---|---|
| **Visibility** | ✅ High | ⚠️ Medium | ❌ Low | ❌ Low |
| **REST purity** | ❌ Violates URI identity | ⚠️ Acceptable | ✅ Clean URIs | ✅ Idiomatic HTTP |
| **Ease of testing** | ✅ Trivial | ✅ Simple | ⚠️ Requires header tools | ⚠️ Requires header tools |
| **Caching** | ✅ Natural separation | ⚠️ Vary on query string | ⚠️ Vary on header | ⚠️ Vary on Accept |
| **Gateway routing** | ✅ Simple path match | ⚠️ Param inspection | ⚠️ Header inspection | ⚠️ Header inspection |
| **Client migration** | ❌ URL change required | ✅ Add/change parameter | ✅ Change header | ✅ Change Accept header |
| **Best for** | Public APIs | Internal APIs | Internal APIs | Hypermedia APIs |

---

## Semantic Versioning for APIs

Semantic Versioning (SemVer) provides a structured numbering scheme that communicates the nature of changes. While most HTTP APIs only expose the **major** version externally (e.g., `/v1/`, `/v2/`), the full SemVer is valuable for internal tracking.

```
MAJOR.MINOR.PATCH

MAJOR — Breaking changes that require consumer updates
MINOR — New features, backward-compatible additions
PATCH — Bug fixes, backward-compatible corrections
```

Expose the precise version via a response header while keeping the URI simple:

```http
HTTP/1.1 200 OK
Content-Type: application/json
X-API-Version: 2.3.1

{
  "id": 42,
  "first_name": "Alice",
  "last_name": "Johnson"
}
```

---

## Identifying Breaking Changes

Clearly defining what constitutes a breaking change is critical. Publish this definition as part of your API governance.

| Change | Breaking? | Explanation |
|---|---|---|
| Remove an endpoint | ✅ Yes | Consumers calling it receive 404 |
| Rename a JSON field | ✅ Yes | Consumers parsing the old name fail |
| Change a field's type (string → integer) | ✅ Yes | Deserialization breaks |
| Remove a field from a response | ✅ Yes | Consumers depending on it fail |
| Make an optional request field required | ✅ Yes | Existing requests missing it are rejected |
| Reduce allowed values for an enum | ✅ Yes | Previously valid requests are rejected |
| Change authentication mechanism | ✅ Yes | Existing credentials stop working |
| Add a new optional request field | ❌ No | Existing requests are unaffected |
| Add a new field to a response | ❌ No | Tolerant readers ignore unknown fields |
| Add a new endpoint | ❌ No | Existing endpoints are unchanged |
| Add a new enum value to a response | ⚠️ Maybe | Consumers with strict parsing may break |

Adding a new enum value is a common source of subtle breakage. Consumers that use exhaustive switch/match statements will fail when they encounter an unknown value. Advise consumers to always include a default case.

---

## Deprecation Strategies

Deprecation is the process of phasing out an API version or endpoint. A well-communicated deprecation strategy respects your consumers and gives them time to migrate.

### Deprecation Timeline

```
Phase 1: Announcement (Month 0)
  └─ Announce in changelog, docs, and API responses
  └─ Provide migration guide

Phase 2: Warning Period (Months 1–6)
  └─ Return Deprecation and Sunset headers
  └─ Log usage — contact heavy users directly
  └─ Send reminders at 3 months and 1 month

Phase 3: Sunset (Month 6+)
  └─ Return 410 Gone for removed endpoints
  └─ Redirect documentation to the new version
```

### Sunset and Deprecation Headers (RFC 8594)

```http
HTTP/1.1 200 OK
Content-Type: application/json
Deprecation: true
Sunset: Sat, 01 Nov 2025 00:00:00 GMT
Link: <https://api.example.com/v2/users/42>; rel="successor-version"

{
  "id": 42,
  "name": "Alice Johnson"
}
```

After the sunset date, return a clear error with migration guidance:

```http
HTTP/1.1 410 Gone
Content-Type: application/json

{
  "error": "gone",
  "message": "API v1 has been retired. Please migrate to v2.",
  "migration_guide": "https://docs.example.com/migration/v1-to-v2"
}
```

---

## Backward Compatibility Patterns

Whenever possible, evolve your API without introducing a new version. These patterns help you make changes that are backward-compatible.

### Additive Changes

The safest evolution strategy is to only **add** — never remove or rename.

```json
// Original response
{ "id": 42, "name": "Alice Johnson", "email": "alice@example.com" }

// Evolved response — new fields added, nothing removed
{
  "id": 42,
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "first_name": "Alice",
  "last_name": "Johnson",
  "avatar_url": "https://cdn.example.com/avatars/42.jpg"
}
```

### Field Aliasing

When you need to rename a field, keep the old name as an alias during a transition period. Both `name` and `full_name` return the same value; document that `name` is deprecated and will be removed.

```json
{
  "id": 42,
  "name": "Alice Johnson",
  "full_name": "Alice Johnson",
  "email": "alice@example.com"
}
```

### Default Values for New Fields

When adding a field that is logically required, provide a sensible server-side default so existing consumers are not forced to send it.

```http
POST /v2/orders HTTP/1.1
Content-Type: application/json

{ "product_id": 101, "quantity": 2 }
```

```http
HTTP/1.1 201 Created

{ "id": 501, "product_id": 101, "quantity": 2, "currency": "USD", "priority": "standard" }
```

The `currency` and `priority` fields have defaults. Consumers that do not send them still receive valid responses.

---

## API Evolution Without Versioning

Some API paradigms and patterns minimize or eliminate the need for explicit versioning.

### GraphQL's Approach

GraphQL avoids versioning by design. Clients request exactly the fields they need, so adding new fields never affects existing queries. Deprecation is handled at the field level:

```graphql
type User {
  name: String @deprecated(reason: "Use firstName and lastName")
  firstName: String
  lastName: String
  email: String
}
```

### Tolerant Reader Pattern

The Tolerant Reader pattern (from Martin Fowler) instructs consumers to:

1. **Ignore unknown fields** — do not fail on unexpected data
2. **Do not depend on field ordering** — parse by name, not position
3. **Use defaults for missing optional fields** — do not fail on absence

```
Strict Reader                     Tolerant Reader
─────────────────                 ─────────────────
Sees unknown field → 💥 Error     Sees unknown field → Ignores it
Field order changes → 💥 Error    Field order changes → No impact
Field missing → 💥 Error          Field missing → Uses default
```

When both producer and consumer follow these principles, many changes that would otherwise be breaking become safe.

---

## Migration Strategies

When a new API version is released, you need a strategy for running both versions simultaneously and routing consumers to the correct one.

### Running Multiple Versions

```
                    ┌──────────────────┐
                    │   API Gateway    │
                    │  /v1/* → v1 svc  │
                    │  /v2/* → v2 svc  │
                    └────────┬─────────┘
               ┌─────────────┼─────────────┐
        ┌──────▼──────┐            ┌───────▼─────┐
        │  v1 Service │            │  v2 Service │
        │  (legacy)   │            │  (current)  │
        └──────┬──────┘            └───────┬─────┘
               └─────────┬─────────────────┘
                  ┌──────▼──────┐
                  │  Shared DB  │
                  └─────────────┘
```

### Request Routing Approaches

| Approach | Description | When to Use |
|---|---|---|
| **Path-based routing** | Gateway routes `/v1/*` and `/v2/*` to different backends | URI path versioning |
| **Header-based routing** | Gateway inspects version header | Header versioning |
| **Adapter layer** | Single service translates between versions internally | Small differences between versions |
| **Branch by abstraction** | Feature flags toggle old/new behavior in one codebase | Gradual rollout of changes |

### Adapter Pattern

When version differences are small, a translation layer avoids duplicating the entire service. The v1 adapter translates requests and responses to the v2 format, while v2 clients talk to the core logic directly.

```
  Client (v1) ──► v1 Adapter ──┐
                                ├──► Core Logic
  Client (v2) ──► v2 API ──────┘
```

---

## Best Practices

1. **Version from day one** — Retrofitting versioning onto an existing API is painful. Start with `/v1/` even if you do not foresee breaking changes.

2. **Minimize supported versions** — Each active version multiplies testing, documentation, and maintenance burden. Aim for two concurrent versions at most.

3. **Batch breaking changes** — Combine breaking changes into a single major version rather than releasing many incremental ones.

4. **Document everything** — Publish a changelog, migration guide, and deprecation timeline with every version.

5. **Use the Sunset header** — Machine-readable deprecation signals let consumers automate migration tracking.

6. **Monitor version usage** — Track which consumers use which versions so you can target communication and assess retirement timing.

7. **Design for backward compatibility first** — Prefer additive changes, the tolerant reader pattern, and field aliasing over new versions.

8. **Automate contract testing** — Use consumer-driven contract testing (e.g., Pact) to catch breaking changes before they ship.

---

## Next Steps

Return to the [API Design Overview](00-OVERVIEW.md) for the full topic map, or revisit [RESTful API Design](01-REST.md), [GraphQL](02-GRAPHQL.md), or [gRPC](03-GRPC.md) to see how versioning concepts apply to each paradigm.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial API versioning documentation |
