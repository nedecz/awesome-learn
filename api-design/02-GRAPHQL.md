# GraphQL API Design

## Table of Contents

1. [Overview](#overview)
2. [What is GraphQL](#what-is-graphql)
3. [Schema Design with SDL](#schema-design-with-sdl)
4. [Queries](#queries)
5. [Mutations](#mutations)
6. [Resolvers](#resolvers)
7. [Subscriptions](#subscriptions)
8. [Pagination](#pagination)
9. [Error Handling](#error-handling)
10. [Authentication and Authorization](#authentication-and-authorization)
11. [Federation and Schema Stitching](#federation-and-schema-stitching)
12. [Performance Considerations](#performance-considerations)
13. [GraphQL vs REST](#graphql-vs-rest)
14. [Best Practices](#best-practices)
15. [Next Steps](#next-steps)
16. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to designing APIs with GraphQL. It covers schema design, query and mutation patterns, resolver implementation, real-time subscriptions, pagination, error handling, security, federation, and performance.

### Target Audience

- **Backend Developers** building GraphQL servers and defining schemas
- **Frontend Developers** consuming GraphQL APIs and writing queries
- **Architects** evaluating GraphQL as an API strategy
- **Tech Leads** establishing conventions for GraphQL adoption

### Scope

- GraphQL fundamentals and differences from REST
- Schema Definition Language — types, interfaces, unions, enums, input types
- Resolver patterns, the N+1 problem, and DataLoader
- Pagination, error handling, authentication, and authorization
- Apollo Federation, query complexity, and persisted queries

---

## What is GraphQL

GraphQL is a **query language for APIs** and a server-side runtime for executing those queries against a typed schema. Developed by Facebook in 2012 and open-sourced in 2015, it lets the **client specify exactly the data it needs** in a single request.

```
┌──────────┐   POST /graphql              ┌──────────────┐
│  Client  │  ─────────────────────────►  │   GraphQL    │
│          │  { query, variables }         │   Server     │
│          │  ◄─────────────────────────  │  ┌────────┐  │
│          │  { data, errors }            │  │ Schema │  │
└──────────┘                              │  └────────┘  │
                                          └──────────────┘
```

**Core Principles**: single endpoint (`/graphql`), client-driven queries, strongly typed schema, hierarchical responses, and built-in introspection.

| Use GraphQL When | Stick with REST When |
|---|---|
| Clients have diverse data needs (mobile vs. web) | You need simple CRUD with uniform responses |
| You want to eliminate over-fetching / under-fetching | HTTP caching at the resource level is critical |
| Your data is a graph of interconnected entities | You serve primarily file uploads or binary data |
| Multiple frontend teams consume the same API | Your consumers expect conventional HTTP semantics |

---

## Schema Design with SDL

The Schema Definition Language (SDL) is the declarative syntax for defining a GraphQL schema.

### Scalar Types

| Scalar | Description | Example |
|---|---|---|
| `Int` | 32-bit signed integer | `42` |
| `Float` | Double-precision floating point | `3.14` |
| `String` | UTF-8 character sequence | `"hello"` |
| `Boolean` | `true` or `false` | `true` |
| `ID` | Unique identifier (serialized as string) | `"abc-123"` |

Custom scalars: `scalar DateTime`, `scalar Email`, `scalar URL`.

### Types, Enums, Interfaces, and Unions

```graphql
type User implements Node {
  id: ID!
  name: String!
  email: String!
  createdAt: DateTime!
  orders: [Order!]!
}

type Order {
  id: ID!
  status: OrderStatus!
  total: Float!
  items: [OrderItem!]!
}

enum OrderStatus { PENDING, PROCESSING, SHIPPED, DELIVERED, CANCELLED }

interface Node { id: ID! }

union SearchResult = User | Product | Order
```

The `!` suffix means **non-null**. Clients resolve union members with inline fragments:

```graphql
query {
  search(term: "Alice") {
    ... on User    { id name email }
    ... on Product { id name price }
    ... on Order   { id status total }
  }
}
```

### Input Types

```graphql
input CreateUserInput {
  name: String!
  email: String!
  role: UserRole = VIEWER
}
```

---

## Queries

Queries are read operations. The client describes the exact shape of the data it needs.

```graphql
query GetUser($userId: ID!) {
  user(id: $userId) {
    id
    name
    email
  }
}
```

```json
{ "data": { "user": { "id": "42", "name": "Alice Johnson", "email": "alice@example.com" } } }
```

### Nested Queries

GraphQL fetches related data in a single request — no multiple round trips:

```graphql
query GetUserWithOrders {
  user(id: "42") {
    name
    orders {
      id
      status
      items { product { name } quantity }
    }
  }
}
```

### Aliases and Fragments

```graphql
fragment UserFields on User { id name email }

query CompareTwoUsers {
  alice: user(id: "42") { ...UserFields }
  bob: user(id: "43") { ...UserFields }
}
```

---

## Mutations

Mutations are write operations. By convention, each mutation accepts a single `input` argument and returns a payload type with the result and any user-facing errors.

```graphql
input CreateOrderInput {
  customerId: ID!
  items: [OrderItemInput!]!
}

type CreateOrderPayload {
  order: Order
  errors: [UserError!]!
}

type UserError { field: String, message: String! }

type Mutation {
  createOrder(input: CreateOrderInput!): CreateOrderPayload!
  cancelOrder(id: ID!): CancelOrderPayload!
}
```

### Example

```graphql
mutation PlaceOrder($input: CreateOrderInput!) {
  createOrder(input: $input) {
    order { id status total }
    errors { field message }
  }
}
```

Variables: `{ "input": { "customerId": "42", "items": [{ "productId": "7", "quantity": 2 }] } }`

Response: `{ "data": { "createOrder": { "order": { "id": "101", "status": "PENDING", "total": 89.97 }, "errors": [] } } }`

---

## Resolvers

Resolvers are functions that populate data for each field. Signature: `resolver(parent, args, context, info)`.

| Parameter | Description |
|---|---|
| `parent` | Result from the parent field's resolver |
| `args` | Arguments passed to the field |
| `context` | Shared per-request state (auth user, data sources, loaders) |
| `info` | AST and schema metadata |

```javascript
const resolvers = {
  Query: {
    user: (_, { id }, { dataSources }) => dataSources.userAPI.getUser(id),
  },
  User: {
    orders: (parent, _, { dataSources }) =>
      dataSources.orderAPI.getByCustomer(parent.id),
  },
};
```

### The N+1 Problem and DataLoader

Fetching a list of users where each triggers a separate DB call for orders produces N+1 queries. **DataLoader** batches and caches loads within a single request tick:

```javascript
const DataLoader = require('dataloader');
const orderLoader = new DataLoader(async (userIds) => {
  const orders = await db.query(
    'SELECT * FROM orders WHERE customer_id = ANY($1)', [userIds]
  );
  return userIds.map(id => orders.filter(o => o.customerId === id));
});
// Resolver: User.orders = (parent) => orderLoader.load(parent.id)
// Without DataLoader: 1 + N queries (e.g., 51). With DataLoader: 1 + 1 (batched).
```

---

## Subscriptions

Subscriptions provide real-time, server-pushed updates via persistent connections (typically WebSocket).

```graphql
type Subscription {
  orderStatusChanged(orderId: ID!): Order!
}
```

```javascript
const { PubSub } = require('graphql-subscriptions');
const pubsub = new PubSub();

const resolvers = {
  Mutation: {
    updateOrderStatus: async (_, { id, status }, { dataSources }) => {
      const order = await dataSources.orderAPI.updateStatus(id, status);
      pubsub.publish(`ORDER_STATUS_${id}`, { orderStatusChanged: order });
      return order;
    },
  },
  Subscription: {
    orderStatusChanged: {
      subscribe: (_, { orderId }) =>
        pubsub.asyncIterableIterator(`ORDER_STATUS_${orderId}`),
    },
  },
};
```

| Transport | Protocol | Notes |
|---|---|---|
| **WebSocket** (`graphql-ws`) | `wss://` | Preferred. Full-duplex persistent connection. |
| **Server-Sent Events** | `https://` | Uni-directional. Works through HTTP proxies. |
| **HTTP Streaming** | `https://` | Multipart responses. Some newer clients support this. |

---

## Pagination

Never return unbounded lists. GraphQL supports two primary strategies.

### Relay-Style Cursor Connections (Recommended)

```graphql
type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int
}

type UserEdge { node: User!, cursor: String! }

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

```graphql
query { users(first: 10, after: "cursor_abc") {
  edges { node { id name } cursor }
  pageInfo { hasNextPage endCursor }
}}
```

### Offset-Based Pagination

Simpler but less robust: `users(page: 1, pageSize: 20): UserPage!`

| Aspect | Cursor-Based | Offset-Based |
|---|---|---|
| **Stability** | Stable across inserts/deletes | Results shift when data changes |
| **Performance** | Consistent at any depth | Degrades at high offsets |
| **Random access** | Not supported | Jump to any page |
| **Complexity** | Higher (opaque cursors) | Lower (simple integers) |

---

## Error Handling

GraphQL responses can contain **both `data` and `errors`**, enabling partial success.

### GraphQL Error Format

```json
{
  "errors": [{ "message": "User not found", "locations": [{ "line": 2, "column": 3 }],
    "path": ["user"], "extensions": { "code": "NOT_FOUND" } }],
  "data": { "user": null }
}
```

| Field | Description |
|---|---|
| `message` | Human-readable error description |
| `locations` | Position in the query where the error occurred |
| `path` | Response field path that triggered the error |
| `extensions` | Custom metadata — error codes, trace IDs, timestamps |

### Partial Responses

If one field fails, other fields still return successfully:

```json
{
  "data": { "user": { "name": "Alice", "orders": null } },
  "errors": [{ "message": "Order service unavailable", "path": ["user", "orders"],
    "extensions": { "code": "SERVICE_UNAVAILABLE" } }]
}
```

### Application-Level Errors in Payloads

For expected business errors, return them in the mutation payload. Keep the top-level `errors` array for unexpected infrastructure failures.

```graphql
type CreateUserPayload { user: User, errors: [UserError!]! }
enum ErrorCode { VALIDATION_FAILED, DUPLICATE_ENTRY, NOT_FOUND, FORBIDDEN }
```

---

## Authentication and Authorization

Authentication is handled at the transport layer; authorization is enforced in resolvers or directives.

```javascript
// Authentication — extract user from JWT into context
const server = new ApolloServer({
  typeDefs, resolvers,
  context: ({ req }) => {
    const token = req.headers.authorization?.replace('Bearer ', '');
    return { user: verifyToken(token) };
  },
});
```

### Resolver-Level Authorization

```javascript
Query: {
  adminDashboard: (_, __, { user }) => {
    if (!user) throw new AuthenticationError('Not authenticated');
    if (user.role !== 'ADMIN') throw new ForbiddenError('Admin access required');
    return getDashboardData();
  },
}
```

### Schema Directives

```graphql
directive @auth(requires: Role!) on FIELD_DEFINITION
enum Role { VIEWER, EDITOR, ADMIN }

type Query {
  publicFeed: [Post!]!
  adminDashboard: DashboardData! @auth(requires: ADMIN)
}
```

---

## Federation and Schema Stitching

Federation allows multiple teams to own and deploy subgraph schemas independently, composed into a single **supergraph** by a gateway.

```
┌─────────────┐
│   Gateway    │  ← Clients send queries here
└──┬───┬───┬──┘
   ▼   ▼   ▼
┌─────┐ ┌──────┐ ┌────────┐
│Users│ │Orders│ │Products│   ← Subgraphs owned by different teams
└─────┘ └──────┘ └────────┘
```

```graphql
# Users subgraph
type User @key(fields: "id") { id: ID!  name: String!  email: String! }

# Orders subgraph — extends User without owning it
type User @key(fields: "id") { id: ID!  orders: [Order!]! }
type Order @key(fields: "id") { id: ID!  status: OrderStatus!  total: Float! }
```

| Directive | Description |
|---|---|
| `@key` | Uniquely identifies an entity across subgraphs |
| `@external` | Marks a field as defined in another subgraph |
| `@requires` | Declares fields from the parent a resolver needs |
| `@provides` | Declares fields this subgraph can resolve for a referenced entity |
| `@shareable` | Allows multiple subgraphs to resolve the same field |

Schema stitching is an older alternative. Federation is preferred for new projects.

---

## Performance Considerations

### Query Complexity Analysis

Assign a cost to each field and reject queries exceeding a threshold:

```
{ users(first: 100) { orders { items { product { reviews } } } } }
Estimated cost: 5400 → exceeds maximum (1000) → 400 Bad Request
```

### Depth Limiting

Restrict maximum nesting depth: `✅ Depth 3: { user { orders { status } } }` vs `❌ Depth 7` — set maximum allowed depth (e.g., 5).

### Persisted Queries

Register queries at build time and send only a hash: `{ "extensions": { "persistedQuery": { "sha256Hash": "abc123..." } } }`. Benefits: reduced bandwidth, query allowlisting, CDN-friendly GET requests.

---

## GraphQL vs REST

| Aspect | REST | GraphQL |
|---|---|---|
| **Endpoints** | Multiple (one per resource) | Single (`/graphql`) |
| **Data fetching** | Server defines response shape | Client specifies exact fields |
| **Over-fetching** | Common — fixed payloads | Eliminated — request only needed fields |
| **Under-fetching** | Multiple round trips | Related data in one query |
| **Versioning** | URL or header (`/v1/`, `/v2/`) | Schema evolution with `@deprecated` |
| **Caching** | HTTP caching (ETags, Cache-Control) | Requires custom caching strategy |
| **File uploads** | Native multipart support | Requires separate spec |
| **Real-time** | SSE, WebSocket (separate setup) | Built-in subscriptions |
| **Error model** | HTTP status codes | `errors` array with partial `data` |
| **Tooling** | Swagger / OpenAPI, Postman | GraphiQL, Apollo Studio, introspection |
| **Learning curve** | Lower — familiar HTTP semantics | Higher — new query language and type system |
| **Best for** | Public APIs, simple CRUD | Complex data graphs, multiple consumers |

---

## Best Practices

### Schema Design

- ✅ Design the schema around your domain, not your database tables
- ✅ Use non-null (`!`) for fields that should always have a value
- ✅ Prefer specific types over generic `String` or `JSON` scalars
- ❌ Do not expose internal identifiers or implementation details

### Naming Conventions

- ✅ `camelCase` for fields, `PascalCase` for types, `SCREAMING_SNAKE_CASE` for enums
- ✅ Prefix mutation inputs with the operation name (`CreateUserInput`)

### Operations

- ✅ Return a payload type from every mutation (not the entity directly)
- ✅ Include an `errors` field in mutation payloads for business-level errors
- ✅ Use variables instead of string interpolation for dynamic values
- ✅ Name all operations — no anonymous queries in production
- ❌ Do not perform side effects in query resolvers

### Security

- ✅ Enforce query depth and complexity limits
- ✅ Use persisted queries to prevent arbitrary query execution
- ✅ Rate-limit by query complexity, not just request count
- ✅ Disable introspection in production environments
- ❌ Do not expose detailed error messages or stack traces to clients

---

## Next Steps

Return to [RESTful API Design](01-REST.md) for a comparison with resource-oriented approaches, or continue to [API Versioning](04-API-VERSIONING.md) to learn how GraphQL handles schema evolution through field deprecation and additive changes.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial GraphQL API design documentation |
