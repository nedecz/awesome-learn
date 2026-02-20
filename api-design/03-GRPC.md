# gRPC API Design

## Table of Contents

1. [Overview](#overview)
2. [What is gRPC](#what-is-grpc)
3. [Protocol Buffers](#protocol-buffers)
4. [Service Definitions](#service-definitions)
5. [Communication Patterns](#communication-patterns)
6. [Error Handling](#error-handling)
7. [Deadlines and Timeouts](#deadlines-and-timeouts)
8. [Interceptors and Middleware](#interceptors-and-middleware)
9. [Load Balancing and Service Discovery](#load-balancing-and-service-discovery)
10. [gRPC-Web for Browser Clients](#grpc-web-for-browser-clients)
11. [gRPC vs REST vs GraphQL](#grpc-vs-rest-vs-graphql)
12. [When to Choose gRPC](#when-to-choose-grpc)
13. [Best Practices](#best-practices)
14. [Next Steps](#next-steps)
15. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to designing APIs with gRPC. It covers Protocol Buffers, service definitions, the four communication patterns, error handling, deadlines, interceptors, load balancing, gRPC-Web, and guidance on when gRPC is the right choice.

### Target Audience

- **Backend Developers** building high-performance internal services
- **Platform Engineers** designing service-to-service communication
- **Architects** evaluating RPC frameworks for polyglot environments
- **Tech Leads** establishing conventions for gRPC adoption across teams

### Scope

- gRPC fundamentals — HTTP/2, binary serialization, and generated code
- Protocol Buffers (proto3) — message types, field numbering, and evolution
- Four RPC patterns — unary, server streaming, client streaming, bidirectional
- Production concerns — error handling, deadlines, interceptors, load balancing
- Cross-technology comparison — gRPC vs REST vs GraphQL

---

## What is gRPC

gRPC is a high-performance, open-source **Remote Procedure Call (RPC) framework** originally developed by Google. It uses **HTTP/2** as its transport protocol and **Protocol Buffers** as its default serialization format, enabling efficient, strongly typed communication between services.

```
┌──────────┐   HTTP/2 (binary frames)     ┌──────────────┐
│  Client   │  ────────────────────────►  │   gRPC       │
│  (Stub)   │  Protobuf request            │   Server     │
│           │  ◄────────────────────────  │  ┌────────┐  │
│           │  Protobuf response           │  │Service │  │
└──────────┘                              │  └────────┘  │
                                          └──────────────┘
```

| Feature | Description |
|---|---|
| **Transport** | HTTP/2 with multiplexed streams, header compression, and flow control |
| **Serialization** | Protocol Buffers — compact binary format, 3–10× smaller than JSON |
| **Code Generation** | `protoc` compiler generates client stubs and server interfaces in 12+ languages |
| **Streaming** | Native support for unary, server streaming, client streaming, and bidirectional |
| **Deadlines** | Built-in deadline propagation across service boundaries |
| **Interceptors** | Middleware chain for logging, auth, metrics, and tracing |

HTTP/2 is foundational: **multiplexing** allows multiple RPCs over a single TCP connection, **binary framing** eliminates text parsing overhead, **HPACK** compresses repeated headers, and **per-stream flow control** prevents fast producers from overwhelming slow consumers.

---

## Protocol Buffers

Protocol Buffers (protobuf) are gRPC's Interface Definition Language (IDL) and serialization format. The `proto3` syntax is the current version.

### Message Types

```protobuf
syntax = "proto3";
package ecommerce.v1;
import "google/protobuf/timestamp.proto";

message Product {
  string id = 1;
  string name = 2;
  string description = 3;
  int32 price_cents = 4;
  ProductCategory category = 5;
  repeated string tags = 6;
  google.protobuf.Timestamp created_at = 7;
}

enum ProductCategory {
  PRODUCT_CATEGORY_UNSPECIFIED = 0;
  PRODUCT_CATEGORY_ELECTRONICS = 1;
  PRODUCT_CATEGORY_CLOTHING = 2;
  PRODUCT_CATEGORY_BOOKS = 3;
}
```

### Field Numbers and Wire Types

Every field has a unique **field number** used for binary encoding. These numbers must never change once deployed.

| Range | Usage |
|---|---|
| 1–15 | Frequently set fields — encoded in one byte |
| 16–2047 | Less common fields — encoded in two bytes |
| 19000–19999 | Reserved by the Protocol Buffers implementation |

| Wire Type | Protobuf Types | Encoding |
|---|---|---|
| 0 (Varint) | `int32`, `int64`, `uint32`, `bool`, `enum` | Variable-length integer |
| 1 (64-bit) | `fixed64`, `sfixed64`, `double` | Fixed 8 bytes |
| 2 (Length-delimited) | `string`, `bytes`, `message`, `repeated` | Length prefix + data |
| 5 (32-bit) | `fixed32`, `sfixed32`, `float` | Fixed 4 bytes |

### Schema Evolution Rules

- ✅ Add new fields with new field numbers
- ✅ Remove fields but **reserve** their numbers: `reserved 6, 8 to 12;`
- ✅ Rename fields freely — only the number matters on the wire
- ❌ Never reuse a field number for a different type
- ❌ Never change a field's wire type

### Nested Messages and Oneofs

```protobuf
message Order {
  string id = 1;
  Customer customer = 2;
  repeated OrderItem items = 3;
  oneof payment_method {
    CreditCard credit_card = 5;
    BankTransfer bank_transfer = 6;
    DigitalWallet digital_wallet = 7;
  }
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  int64 unit_price_cents = 3;
}
```

`oneof` ensures at most one of the enclosed fields is set — useful for polymorphic data.

---

## Service Definitions

Services define the RPC methods a server exposes. The `protoc` compiler generates both the client stub and the server interface from a single `.proto` file.

```protobuf
syntax = "proto3";
package ecommerce.v1;

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc ListOrders(ListOrdersRequest) returns (stream Order);
  rpc UploadLineItems(stream OrderItem) returns (UploadSummary);
  rpc OrderChat(stream ChatMessage) returns (stream ChatMessage);
}

message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
  string idempotency_key = 3;
}

message CreateOrderResponse { Order order = 1; }
message GetOrderRequest { string id = 1; }
message ListOrdersRequest {
  string customer_id = 1;
  int32 page_size = 2;
  string page_token = 3;
}
message UploadSummary { int32 items_received = 1; int32 items_accepted = 2; }
message ChatMessage { string order_id = 1; string sender = 2; string text = 3; }
```

Generate code with: `protoc --go_out=. --go-grpc_out=. proto/ecommerce/v1/order.proto`

---

## Communication Patterns

gRPC supports four distinct RPC patterns, each suited to different interaction models.

### Unary RPC

One request, one response — equivalent to a traditional function call.

```go
func (s *server) GetOrder(ctx context.Context, req *pb.GetOrderRequest) (*pb.Order, error) {
    order, err := s.store.FindByID(ctx, req.GetId())
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "order %s not found", req.GetId())
    }
    return order, nil
}
```

```python
# Client call (Python)
channel = grpc.insecure_channel("localhost:50051")
stub = order_pb2_grpc.OrderServiceStub(channel)
response = stub.GetOrder(order_pb2.GetOrderRequest(id="order-123"))
```

### Server Streaming RPC

The server sends a stream of messages for a single request. Ideal for paginated data or real-time feeds.

```go
func (s *server) ListOrders(req *pb.ListOrdersRequest, stream pb.OrderService_ListOrdersServer) error {
    orders, err := s.store.FindByCustomer(stream.Context(), req.GetCustomerId())
    if err != nil {
        return status.Errorf(codes.Internal, "query failed: %v", err)
    }
    for _, order := range orders {
        if err := stream.Send(order); err != nil {
            return err
        }
    }
    return nil
}
```

### Client Streaming RPC

The client sends a stream of messages and receives a single response. Suited for batch uploads or aggregation.

```go
func (s *server) UploadLineItems(stream pb.OrderService_UploadLineItemsServer) error {
    var received, accepted int32
    for {
        item, err := stream.Recv()
        if err == io.EOF {
            return stream.SendAndClose(&pb.UploadSummary{
                ItemsReceived: received, ItemsAccepted: accepted,
            })
        }
        if err != nil { return err }
        received++
        if s.validate(item) { accepted++ }
    }
}
```

### Bidirectional Streaming RPC

Both sides stream messages independently — neither needs to wait for the other.

```go
func (s *server) OrderChat(stream pb.OrderService_OrderChatServer) error {
    for {
        msg, err := stream.Recv()
        if err == io.EOF { return nil }
        if err != nil { return err }
        reply := &pb.ChatMessage{
            OrderId: msg.GetOrderId(), Sender: "support", Text: "Received: " + msg.GetText(),
        }
        if err := stream.Send(reply); err != nil { return err }
    }
}
```

| Pattern | Request | Response | Use Cases |
|---|---|---|---|
| **Unary** | Single | Single | CRUD operations, simple lookups |
| **Server streaming** | Single | Stream | Feeds, paginated lists, log tailing |
| **Client streaming** | Stream | Single | File uploads, batch ingestion, aggregation |
| **Bidirectional** | Stream | Stream | Chat, collaborative editing, real-time sync |

---

## Error Handling

gRPC uses a structured error model with **status codes**, **messages**, and optional **rich error details**.

### gRPC Status Codes

| Code | Name | HTTP Equivalent | Description |
|---|---|---|---|
| 0 | `OK` | 200 | Success |
| 1 | `CANCELLED` | 499 | Operation cancelled by the caller |
| 2 | `UNKNOWN` | 500 | Unknown error |
| 3 | `INVALID_ARGUMENT` | 400 | Client sent an invalid argument |
| 4 | `DEADLINE_EXCEEDED` | 504 | Deadline expired before completion |
| 5 | `NOT_FOUND` | 404 | Requested resource does not exist |
| 6 | `ALREADY_EXISTS` | 409 | Resource already exists |
| 7 | `PERMISSION_DENIED` | 403 | Caller lacks permission |
| 8 | `RESOURCE_EXHAUSTED` | 429 | Rate limit or quota exceeded |
| 9 | `FAILED_PRECONDITION` | 400 | System not in required state |
| 10 | `ABORTED` | 409 | Concurrency conflict |
| 11 | `OUT_OF_RANGE` | 400 | Value outside valid range |
| 12 | `UNIMPLEMENTED` | 501 | Method not implemented |
| 13 | `INTERNAL` | 500 | Internal server error |
| 14 | `UNAVAILABLE` | 503 | Service temporarily unavailable — retry |
| 16 | `UNAUTHENTICATED` | 401 | Missing or invalid authentication |

### Rich Error Details

The `google.rpc.Status` model supports structured detail messages for richer diagnostics:

```go
st := status.New(codes.InvalidArgument, "invalid order request")
st, _ = st.WithDetails(
    &errdetails.BadRequest{
        FieldViolations: []*errdetails.BadRequest_FieldViolation{
            {Field: "customer_id", Description: "must not be empty"},
            {Field: "items", Description: "at least one item is required"},
        },
    },
)
return nil, st.Err()
```

---

## Deadlines and Timeouts

gRPC deadlines specify the **absolute point in time** by which an RPC must complete. Unlike relative timeouts, deadlines propagate across service boundaries automatically.

```
Client (deadline: 5s) ──► Service A (remaining: 4.2s) ──► Service B (remaining: 3.1s)
```

```go
// Client sets a 5-second deadline (Go)
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

order, err := client.GetOrder(ctx, &pb.GetOrderRequest{Id: "order-123"})
if st, ok := status.FromError(err); ok && st.Code() == codes.DeadlineExceeded {
    log.Println("Request timed out")
}
```

```python
# Python equivalent
response = stub.GetOrder(request, timeout=5.0)
```

**Best practice**: always set deadlines. An RPC without a deadline can consume resources indefinitely.

---

## Interceptors and Middleware

Interceptors are gRPC's equivalent of middleware — they wrap RPC handlers to inject cross-cutting concerns.

### Server Interceptors

```go
func loggingInterceptor(
    ctx context.Context, req interface{},
    info *grpc.UnaryServerInfo, handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    log.Printf("method=%s duration=%s error=%v", info.FullMethod, time.Since(start), err)
    return resp, err
}

server := grpc.NewServer(
    grpc.ChainUnaryInterceptor(loggingInterceptor, authInterceptor, metricsInterceptor),
    grpc.ChainStreamInterceptor(streamLoggingInterceptor),
)
```

### Client Interceptors

```go
func authClientInterceptor(
    ctx context.Context, method string, req, reply interface{},
    cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption,
) error {
    ctx = metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer "+getToken())
    return invoker(ctx, method, req, reply, cc, opts...)
}
```

| Concern | Server Interceptor | Client Interceptor |
|---|---|---|
| **Logging** | Log method, duration, status code | Log outgoing calls and responses |
| **Authentication** | Validate tokens from metadata | Attach tokens to outgoing metadata |
| **Metrics** | Record latency, error rate, throughput | Record call latency, retries |
| **Tracing** | Extract and propagate trace context | Inject trace headers |
| **Rate Limiting** | Enforce per-client or per-method limits | — |
| **Retry** | — | Retry on `UNAVAILABLE` with backoff |

---

## Load Balancing and Service Discovery

gRPC supports both **proxy-based** (L7) and **client-side** load balancing.

```
Proxy-based:  Client ──► L7 Proxy (Envoy / Linkerd) ──► Server 1, 2, 3
Client-side:  Client ──► (resolves DNS) ──► Server 1, 2, 3 directly
```

```go
// Client-side round-robin (Go)
conn, _ := grpc.Dial(
    "dns:///orders.internal.svc:50051",
    grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`),
)
```

| Strategy | How It Works | Pros | Cons |
|---|---|---|---|
| **Pick First** | Use the first resolved address | Simplest | No balancing |
| **Round Robin** | Rotate through all addresses | Even distribution | Ignores server load |
| **Weighted** | Distribute by server capacity | Load-aware | Requires health reporting |
| **Look-Aside (xDS)** | External balancer directs traffic | Fine-grained control | Additional infrastructure |

In Kubernetes, **headless services** (`clusterIP: None`) expose individual pod IPs for client-side balancing. For proxy-based approaches, service meshes like **Istio** and **Linkerd** natively understand gRPC and can load-balance at the request level.

---

## gRPC-Web for Browser Clients

Browsers cannot use native gRPC because they lack direct HTTP/2 frame control. **gRPC-Web** bridges this gap with a translation proxy.

```
Browser (JS/TS) ──[gRPC-Web over HTTP/1.1]──► Envoy Proxy ──[native gRPC]──► gRPC Server
```

```typescript
import { OrderServiceClient } from "./generated/order_grpc_web_pb";
import { GetOrderRequest } from "./generated/order_pb";

const client = new OrderServiceClient("https://api.example.com");
const request = new GetOrderRequest();
request.setId("order-123");

client.getOrder(request, {}, (err, response) => {
  if (err) { console.error(`Error: ${err.code} — ${err.message}`); return; }
  console.log(`Order status: ${response.getStatus()}`);
});
```

| Feature | Native gRPC | gRPC-Web |
|---|---|---|
| **Unary RPC** | ✅ | ✅ |
| **Server streaming** | ✅ | ✅ |
| **Client streaming** | ✅ | ❌ |
| **Bidirectional streaming** | ✅ | ❌ |
| **Transport** | HTTP/2 only | HTTP/1.1 or HTTP/2 |
| **Proxy required** | No | Yes (Envoy, grpc-web-proxy) |

---

## gRPC vs REST vs GraphQL

| Aspect | REST | GraphQL | gRPC |
|---|---|---|---|
| **Protocol** | HTTP/1.1 or HTTP/2 | HTTP/1.1 or HTTP/2 | HTTP/2 |
| **Serialization** | JSON (text) | JSON (text) | Protocol Buffers (binary) |
| **Schema / Contract** | OpenAPI (optional) | SDL (required) | `.proto` files (required) |
| **Code Generation** | Optional | Optional | Built-in (`protoc`) |
| **Payload Size** | Larger (text + field names) | Medium (client-selected) | Smallest (binary, no field names) |
| **Streaming** | Limited (SSE, WebSocket) | Subscriptions (WebSocket) | Native (4 patterns) |
| **Browser Support** | Native | Native | Requires gRPC-Web proxy |
| **Caching** | HTTP caching (ETags, CDN) | Custom caching required | No built-in caching |
| **Error Model** | HTTP status codes | `errors` array with partial data | Status codes + rich error details |
| **Best For** | Public APIs, web clients | Complex data graphs, multi-consumer | Internal services, high-throughput RPC |

---

## When to Choose gRPC

### Ideal Use Cases

- **Service-to-service communication** — binary protocol and code generation reduce latency and eliminate serialization bugs between microservices
- **Polyglot environments** — a single `.proto` file generates idiomatic clients in Go, Java, Python, C++, Rust, and more
- **Real-time streaming** — native support for all four streaming patterns
- **High-throughput systems** — multiplexed HTTP/2 and compact payloads handle thousands of concurrent RPCs
- **Strict contracts** — `.proto` schema enforces type safety and backward compatibility across teams

### When to Prefer Alternatives

| Scenario | Better Choice | Reason |
|---|---|---|
| Public-facing web API | REST | Universal browser support, HTTP caching |
| Diverse frontend data needs | GraphQL | Clients select exactly the fields they need |
| Simple CRUD with few consumers | REST | Lower complexity, easier onboarding |
| Browser-only real-time features | WebSocket / SSE | No proxy requirement, simpler stack |

### Decision Checklist

- ✅ Communication is **service-to-service** (backend ↔ backend)
- ✅ You need **streaming** (server, client, or bidirectional)
- ✅ Performance and **low latency** are critical requirements
- ✅ Teams use **multiple programming languages**
- ✅ You already operate a **service mesh** that natively supports gRPC
- ❌ Primary consumers are **web browsers** without a gRPC-Web proxy
- ❌ You need **HTTP-level caching** (CDN, ETags, Cache-Control)

---

## Best Practices

### API Design

- ✅ Use **package versioning** in proto files: `package myservice.v1;`
- ✅ Define request and response messages for every RPC — even if the body is empty
- ✅ Include an `idempotency_key` field for mutating operations
- ✅ Use `FieldMask` for partial updates instead of overwriting entire resources
- ❌ Do not use `float`/`double` for monetary values — use `int64` cents or a `Money` message

### Production Readiness

- ✅ Always set **deadlines** — never send an RPC without one
- ✅ Implement **health checking** (`grpc.health.v1.Health`) for load balancer integration
- ✅ Use **TLS** in production — mutual TLS (mTLS) for zero-trust environments
- ✅ Enable **reflection** in development for tools like `grpcurl` and `grpcui`
- ✅ Implement **graceful shutdown** to drain in-flight RPCs before terminating
- ❌ Do not expose raw internal errors to clients — map to appropriate status codes

---

## Next Steps

Return to [RESTful API Design](01-REST.md) for resource-oriented approaches, or review [GraphQL API Design](02-GRAPHQL.md) for client-driven query patterns. Compare the three technologies using the [comparison table](#grpc-vs-rest-vs-graphql) to determine the best fit for your system.

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial gRPC API design documentation |
