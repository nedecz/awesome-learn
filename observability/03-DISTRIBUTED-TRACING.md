# Distributed Tracing

## Table of Contents

1. [What is Distributed Tracing](#what-is-distributed-tracing)
2. [Core Concepts](#core-concepts)
3. [Trace Across Microservices](#trace-across-microservices)
4. [W3C Trace Context Standard](#w3c-trace-context-standard)
5. [Jaeger](#jaeger)
6. [Zipkin](#zipkin)
7. [Sampling Strategies](#sampling-strategies)
8. [Instrumentation Examples](#instrumentation-examples)
9. [Context Propagation](#context-propagation)
10. [Trace Analysis and Finding Bottlenecks](#trace-analysis-and-finding-bottlenecks)
11. [Next Steps](#next-steps)

---

## What is Distributed Tracing

### The Problem It Solves

In a monolithic application, when a request is slow or fails, you look at a single set of logs and a stack trace. You can see exactly what happened. In a distributed system, a single user request might touch dozens of services, each with its own logs and metrics, running on different machines.

**Without tracing**, debugging a slow checkout looks like this:

```
User reports: "Checkout is slow"

Engineer checks:
  - api-gateway logs     → Request received, forwarded to order-service (200ms total)
  - order-service logs   → Order created, called payment-service  (120ms)
  - payment-service logs → Payment processed, called fraud-service (???ms)
  - fraud-service logs   → Where are these logs? Different log system
  - database logs        → Hundreds of queries, can't tell which request

Result: 2 hours later, still not sure which service was slow
```

**With distributed tracing**, the same investigation takes minutes:

```
User reports: "Checkout is slow"

Engineer opens Jaeger UI:
  - Search by user_id or request_id
  - Click trace → see full waterfall diagram
  - Fraud-service DB call: 3500ms  ← FOUND IT
  - Missing index on fraud_checks.user_id

Result: 5 minutes, root cause identified
```

### Without Tracing vs. With Tracing

```
WITHOUT TRACING — Request across 4 services
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  [Client]         [API Gateway]      [Order Svc]      [Payment Svc]      [DB]
      │                 │                 │                 │               │
      │──── POST ───────►│                 │                 │               │
      │                 │─── forward ─────►│                 │               │
      │                 │                 │─── pay ──────────►│               │
      │                 │                 │                 │──── query ────►│
      │                 │                 │                 │◄─── result ────│
      │                 │                 │◄─── result ──────│               │
      │◄─── response ───│◄─── response ───│                 │               │

  Problem: Each service has its own logs. No way to correlate them.
  "Which log line in payment-service belongs to THIS checkout request?"


WITH DISTRIBUTED TRACING — Same request, now observable
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736

  [Client]         [API Gateway]      [Order Svc]      [Payment Svc]      [DB]
      │                 │                 │                 │               │
      │──── POST ───────►│ SpanA starts    │                 │               │
      │                 │─── forward ─────►│ SpanB starts    │               │
      │                 │   (traceID+spanA)│─── pay ──────────►│ SpanC starts  │
      │                 │                 │   (traceID+spanB)│──── query ────►│ SpanD starts
      │                 │                 │                 │◄─── result ────│ SpanD ends
      │                 │                 │◄─── result ──────│ SpanC ends     │
      │                 │◄─── response ───│ SpanB ends      │               │
      │◄─── response ───│ SpanA ends      │                 │               │

  All 4 spans share Trace ID → Jaeger assembles them into one timeline
```

---

## Core Concepts

### Trace

A **trace** represents the complete journey of a single request as it flows through a distributed system. It is the top-level container for all the work done to serve one request.

```
  Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736
  ┌─────────────────────────────────────────────────────────────────────┐
  │                          TRACE                                      │
  │                                                                     │
  │  ┌────────────────────────────────────────────────────────────┐    │
  │  │  Span A: POST /checkout  (api-gateway)          195ms     │    │
  │  │  ┌──────────────────────────────────────────────────────┐  │    │
  │  │  │  Span B: createOrder  (order-service)       160ms    │  │    │
  │  │  │  ┌────────────────────────────────────────────────┐  │  │    │
  │  │  │  │  Span C: processPayment (payment-service) 120ms│  │  │    │
  │  │  │  │  ┌──────────────────────────────────────────┐  │  │  │    │
  │  │  │  │  │  Span D: SELECT fraud_checks  (postgres) │  │  │  │    │
  │  │  │  │  │                               80ms       │  │  │  │    │
  │  │  │  │  └──────────────────────────────────────────┘  │  │  │    │
  │  │  │  └────────────────────────────────────────────────┘  │  │    │
  │  │  └──────────────────────────────────────────────────────┘  │    │
  │  └────────────────────────────────────────────────────────────┘    │
  └─────────────────────────────────────────────────────────────────────┘
```

### Span

A **span** represents a single named, timed operation. It is the building block of a trace.

Every span contains:
- **Operation name**: A human-readable description (e.g., `"HTTP POST /checkout"`, `"db.query"`)
- **Trace ID**: The ID of the trace this span belongs to (same for all spans in a trace)
- **Span ID**: A unique identifier for this specific span
- **Parent Span ID**: The ID of the parent span (absent for root spans)
- **Start time**: When the operation began
- **Duration**: How long the operation took
- **Attributes** (tags): Key-value metadata (e.g., `http.method="POST"`, `db.statement="SELECT..."`)
- **Events** (logs): Timestamped log messages attached to the span
- **Status**: OK, ERROR, or UNSET
- **Links**: References to other spans (for async/batch operations)

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                              SPAN                                    │
  │                                                                      │
  │  name:        "HTTP POST /api/checkout"                              │
  │  trace_id:    4bf92f3577b34da6a3ce929d0e0e4736                       │
  │  span_id:     00f067aa0ba902b7                                       │
  │  parent_id:   (none — this is the root span)                         │
  │  start_time:  2025-01-15T14:32:01.000Z                               │
  │  duration:    195ms                                                  │
  │  status:      OK                                                     │
  │                                                                      │
  │  Attributes:                                                         │
  │    http.method:        POST                                          │
  │    http.url:           https://api.example.com/checkout              │
  │    http.status_code:   200                                           │
  │    user.id:            usr_abc123                                    │
  │                                                                      │
  │  Events:                                                             │
  │    14:32:01.050  "Auth token validated"                              │
  │    14:32:01.100  "Routing to order-service"                          │
  └──────────────────────────────────────────────────────────────────────┘
```

### Trace ID

A **Trace ID** is a globally unique identifier (128-bit, typically represented as a 32-character hex string) assigned to a trace when it begins. Every span in the same trace carries the same Trace ID. This is the key that allows Jaeger or Zipkin to assemble spans from multiple services into one coherent trace.

```
  Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736
            └────────────────────────────────┘
                128 bits = 32 hex characters
```

### Span ID

A **Span ID** is a unique identifier (64-bit, 16 hex characters) assigned to each individual span. It is unique within a trace. Span IDs are used to link parent and child spans together.

```
  Span ID: 00f067aa0ba902b7
           └────────────────┘
            64 bits = 16 hex characters
```

### Parent Span / Child Span

Spans form a tree structure. The **root span** has no parent. Every other span has a **parent span ID** pointing to the span that created it.

```
  Root Span (api-gateway, span_id=AAAA, parent=none)
    │
    └─► Child Span (order-service, span_id=BBBB, parent=AAAA)
            │
            ├─► Child Span (payment-service, span_id=CCCC, parent=BBBB)
            │       │
            │       └─► Child Span (postgres, span_id=DDDD, parent=CCCC)
            │
            └─► Child Span (inventory-service, span_id=EEEE, parent=BBBB)
```

### Baggage

**Baggage** is a set of key-value pairs that are propagated alongside the trace context across all service boundaries. Baggage is intended for metadata that every downstream service might need (e.g., `tenant_id`, `experiment_id`), but it has a performance cost because it travels in every request header.

```
  Use baggage for:
  ✅ tenant_id (needed in every service for multi-tenancy)
  ✅ feature_flag (A/B test ID needed for logging decisions)

  Do NOT use baggage for:
  ❌ Large payloads (grows every request header)
  ❌ High-cardinality values (user_id of every user = large headers)
  ❌ Security-sensitive data (visible in headers, logs)
```

### Context Propagation

**Context propagation** is the mechanism by which trace context (trace ID, span ID, sampling decision, baggage) is passed from one service to the next. In HTTP, this is done via request headers. The W3C Trace Context specification defines the standard format.

```
  Service A creates a span and injects context into outgoing headers:
  ┌─────────────────────────────────────────────────────────────────┐
  │  HTTP Request Headers (outgoing from Service A)                 │
  │                                                                 │
  │  traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
  │  tracestate:  vendor1=opaquevalue1                              │
  └─────────────────────────────────────────────────────────────────┘

  Service B receives the request and extracts context:
    → Creates a new child span with parent=00f067aa0ba902b7
    → Continues propagating to Service C
```

---

## Trace Across Microservices

A detailed timing diagram showing a checkout request flowing through four services:

```
  Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736

  Time (ms): 0        50       100      150      200
             │        │        │        │        │
             ├────────────────────────────────────┤
             │            API Gateway             │  (0–195ms, total: 195ms)
             │  span_id: AAAA  parent: (none)     │
             │                                    │
             │  ├─────────────────────────────┤   │
             │  │       Order Service         │   │  (5–165ms, total: 160ms)
             │  │  span_id: BBBB parent: AAAA │   │
             │  │                             │   │
             │  │  ├──────────────────────┤   │   │
             │  │  │   Payment Service    │   │   │  (10–130ms, total: 120ms)
             │  │  │ span_id:CCCC p:BBBB  │   │   │
             │  │  │                      │   │   │
             │  │  │  ├─────────────┤     │   │   │
             │  │  │  │  Postgres   │     │   │   │  (15–95ms, total: 80ms)
             │  │  │  │ span:DDDD   │     │   │   │
             │  │  │  │ p:CCCC      │     │   │   │
             │  │  │  │ SELECT      │     │   │   │
             │  │  │  │ fraud_checks│     │   │   │
             │  │  │  ├─────────────┘     │   │   │
             │  │  │                      │   │   │
             │  │  ├──────────────────────┘   │   │
             │  │                             │   │
             │  │  ├────────────────┤         │   │
             │  │  │  Inventory Svc │         │   │  (130–160ms, total: 30ms)
             │  │  │ span:EEEE p:BB │         │   │
             │  │  ├────────────────┘         │   │
             │  │                             │   │
             │  ├─────────────────────────────┘   │
             │                                    │
             ├────────────────────────────────────┘

  Span Breakdown:
  ┌────────────────────┬────────────┬──────────┬──────────────────────────────────────────────┐
  │ Service            │ Duration   │ % of req │ Notes                                        │
  ├────────────────────┼────────────┼──────────┼──────────────────────────────────────────────┤
  │ API Gateway        │ 195ms      │ 100%     │ Root span — includes all downstream time     │
  │ Order Service      │ 160ms      │  82%     │ Most time in Payment + Inventory calls       │
  │ Payment Service    │ 120ms      │  62%     │ Mostly waiting on database                  │
  │ Postgres (fraud)   │  80ms      │  41%     │ ← BOTTLENECK: missing index                 │
  │ Inventory Service  │  30ms      │  15%     │ Runs in parallel with payment return         │
  └────────────────────┴────────────┴──────────┴──────────────────────────────────────────────┘
```

---

## W3C Trace Context Standard

The [W3C Trace Context](https://www.w3.org/TR/trace-context/) specification defines standard HTTP headers for trace propagation, enabling interoperability across different tracing vendors and libraries.

### traceparent Header

The `traceparent` header carries the trace ID, span ID, and trace flags.

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  │                                │                │
             │  │                                │                └── Flags (8-bit, 01=sampled)
             │  │                                └── Parent Span ID (64-bit / 16 hex chars)
             │  └── Trace ID (128-bit / 32 hex chars)
             └── Version (always "00" currently)
```

| Field | Size | Description |
|---|---|---|
| **version** | 2 hex chars | Always `00` in the current spec |
| **trace-id** | 32 hex chars | 128-bit trace identifier |
| **parent-id** | 16 hex chars | 64-bit span ID of the caller's span |
| **trace-flags** | 2 hex chars | Bit field; `01` = sampled, `00` = not sampled |

### tracestate Header

The `tracestate` header carries vendor-specific state alongside the standard `traceparent`. Multiple vendors can add entries.

```
tracestate: jaeger=3a2b1c,datadog=s:1;p:00f067aa0ba902b7
```

### Code Examples

```go
// Go — injecting W3C trace context into outgoing HTTP request
import (
    "net/http"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/propagation"
)

func callDownstream(ctx context.Context, url string) (*http.Response, error) {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)

    // Inject trace context into HTTP headers (traceparent, tracestate)
    otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))

    // req.Header now contains:
    // traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

    return http.DefaultClient.Do(req)
}

// Go — extracting trace context from incoming HTTP request
func handleRequest(w http.ResponseWriter, r *http.Request) {
    // Extract incoming trace context from headers
    ctx := otel.GetTextMapPropagator().Extract(r.Context(), propagation.HeaderCarrier(r.Header))

    // Start a new child span from the extracted parent context
    tracer := otel.Tracer("order-service")
    ctx, span := tracer.Start(ctx, "handleCheckout")
    defer span.End()

    // All subsequent spans created with this ctx will be children
    processOrder(ctx)
}
```

---

## Jaeger

Jaeger is an open-source, end-to-end distributed tracing system released by Uber Technologies and now a CNCF Graduated project.

### Architecture Diagram

```
                        ┌──────────────────────────────────────────────────────────┐
                        │                    Jaeger Backend                        │
                        │                                                          │
  ┌──────────────┐      │  ┌──────────────┐    ┌─────────────┐    ┌────────────┐  │
  │  Application │      │  │    Jaeger    │    │    Jaeger   │    │   Jaeger   │  │
  │              │      │  │  Collector   │───►│   Storage   │◄───│   Query    │  │
  │  OTel SDK    │─────►│  │              │    │             │    │            │  │
  │  (OTLP/gRPC) │      │  │  Validates & │    │  Cassandra  │    │  REST API  │  │
  └──────────────┘      │  │  stores      │    │  Elasticsearch    │  & UI      │  │
                        │  │  spans       │    │  Badger(dev)│    │            │  │
                        │  └──────────────┘    └─────────────┘    └────────────┘  │
                        │                                               │          │
                        └───────────────────────────────────────────────┼──────────┘
                                                                        │
                                                                        ▼
                                                                 ┌────────────────┐
                                                                 │  Jaeger UI     │
                                                                 │  :16686        │
                                                                 └────────────────┘
```

| Component | Role |
|---|---|
| **Jaeger Client / OTel SDK** | Instruments your application code; creates and exports spans |
| **Jaeger Collector** | Receives spans over OTLP/gRPC or Thrift, validates, and writes to storage |
| **Storage Backend** | Persists spans; Cassandra and Elasticsearch for production; Badger for dev |
| **Jaeger Query** | Serves the API and UI for searching and viewing traces |
| **Jaeger UI** | Web interface for trace exploration, comparison, and DAG views |

### Deployment Options

```bash
# Option 1: All-in-one (development only — no persistence)
docker run -d --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \   # Jaeger UI
  -p 4317:4317 \     # OTLP gRPC
  -p 4318:4318 \     # OTLP HTTP
  jaegertracing/all-in-one:1.52
```

```yaml
# Option 2: Production on Kubernetes using Jaeger Operator
# Install the Jaeger Operator first:
kubectl create namespace observability
kubectl apply -f https://github.com/jaegertracing/jaeger-operator/releases/latest/download/jaeger-operator.yaml \
  -n observability

# Then create a Jaeger instance with Elasticsearch storage:
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-production
  namespace: observability
spec:
  strategy: production   # production = separate collector, query, and storage deployments

  collector:
    replicas: 3
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi

  storage:
    type: elasticsearch
    options:
      es:
        server-urls: https://elasticsearch:9200
        index-prefix: jaeger
        tls:
          skip-host-verify: false
          ca: /es/certificates/ca.crt
    secretName: jaeger-es-secret   # Contains ES username/password

  query:
    replicas: 2
    ingress:
      enabled: true
      hosts:
        - jaeger.internal.example.com
```

### Sampling Configuration

Sampling determines which traces are actually recorded. Recording 100% of traces is cost-prohibitive at scale.

```yaml
# Jaeger sampling strategies (served by Jaeger's sampling server)
# GET http://jaeger-collector:5778/sampling?service=order-service

{
  "default_strategy": {
    "type": "probabilistic",
    "param": 0.1           # Sample 10% of all traces by default
  },
  "per_operation_strategies": [
    {
      "operation": "HTTP GET /health",
      "type": "const",
      "param": 0            # Never sample health checks
    },
    {
      "operation": "HTTP POST /api/orders",
      "type": "ratelimiting",
      "param": 100          # Maximum 100 traces per second for this operation
    },
    {
      "operation": "processPayment",
      "type": "probabilistic",
      "param": 1.0          # Always sample payment operations
    }
  ],
  "service_strategies": [
    {
      "service": "order-service",
      "type": "probabilistic",
      "param": 0.5          # Sample 50% of order-service traces
    }
  ]
}
```

| Strategy | Description | Use When |
|---|---|---|
| **Constant (`const`)** | `param=1` = always sample; `param=0` = never | Health checks (never), critical operations (always) |
| **Probabilistic** | `param=0.1` = sample 10% randomly | Default; adjust based on traffic volume |
| **Rate Limiting** | `param=100` = max 100 traces/second | High-traffic services where you want a steady sample rate |
| **Remote** | Strategy served by Jaeger sampling server | Dynamic, centrally controlled; adjust without redeploying |
| **Adaptive** | Adjusts rate per operation to hit a target volume | High-traffic, heterogeneous services (available in some collectors) |

### Jaeger Client Configuration (Go)

```go
// tracing.go — initialize OpenTelemetry with Jaeger exporter
package tracing

import (
    "context"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
    "google.golang.org/grpc"
)

func InitTracer(serviceName, jaegerEndpoint string) (func(context.Context) error, error) {
    // Create OTLP gRPC exporter (sends to Jaeger Collector)
    exporter, err := otlptracegrpc.New(
        context.Background(),
        otlptracegrpc.WithEndpoint(jaegerEndpoint),   // e.g., "jaeger:4317"
        otlptracegrpc.WithInsecure(),
        otlptracegrpc.WithDialOption(grpc.WithBlock()),
    )
    if err != nil {
        return nil, err
    }

    // Define the resource (service metadata attached to all spans)
    res := resource.NewWithAttributes(
        semconv.SchemaURL,
        semconv.ServiceName(serviceName),
        semconv.ServiceVersion("1.2.3"),
        semconv.DeploymentEnvironment("production"),
    )

    // Create the TracerProvider with probabilistic sampling
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(
            sdktrace.ParentBased(       // Respect parent's sampling decision
                sdktrace.TraceIDRatioBased(0.1),  // 10% sampling for new traces
            ),
        ),
    )

    // Register globally so all instrumentation libraries can use it
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},  // W3C Trace Context (traceparent/tracestate)
        propagation.Baggage{},       // W3C Baggage
    ))

    // Return shutdown function to flush remaining spans on exit
    return tp.Shutdown, nil
}
```

### Jaeger UI Features

| Feature | Description |
|---|---|
| **Trace search** | Find traces by service, operation, tags, min/max duration, time range |
| **Trace timeline** | Waterfall view showing all spans with their timing and nesting |
| **Span details** | Click a span to see all attributes, events, and logs |
| **Service dependency graph** | Automatically generated DAG showing service call relationships |
| **Trace comparison** | Compare two traces side-by-side (useful for before/after performance work) |
| **Deep link** | Every trace and span has a permanent URL for sharing with teammates |
| **Critical path** | Highlights the critical path — the sequence of spans that determined total duration |

---

## Zipkin

Zipkin is a distributed tracing system open-sourced by Twitter. It is simpler than Jaeger and has strong compatibility with many language libraries.

### Architecture

```
  Applications          ┌──────────────────────────────────────────────┐
  (instrumented         │                  Zipkin                      │
   with Zipkin libs)    │                                              │
                        │  ┌────────────┐    ┌────────────────────┐   │
  App A (Java) ────────►│  │            │    │                    │   │
  App B (Go)   ────────►│  │ Collector  │───►│  Storage Backend   │   │
  App C (Node) ────────►│  │            │    │                    │   │
                        │  │ HTTP/9411  │    │ In-memory (dev)    │   │
                        │  │ Kafka      │    │ MySQL              │   │
                        │  │ AMQP       │    │ Cassandra          │   │
                        │  └────────────┘    │ Elasticsearch      │   │
                        │                    └──────────┬─────────┘   │
                        │                               │              │
                        │                    ┌──────────▼─────────┐   │
                        │                    │   API  + UI :9411   │   │
                        │                    └────────────────────┘   │
                        └──────────────────────────────────────────────┘
```

### Docker Compose Deployment

```yaml
# docker-compose.yml — Zipkin with Elasticsearch storage
version: "3.8"
services:
  zipkin:
    image: openzipkin/zipkin:3
    container_name: zipkin
    environment:
      - STORAGE_TYPE=elasticsearch
      - ES_HOSTS=http://elasticsearch:9200
    ports:
      - "9411:9411"   # Zipkin HTTP API and UI
    depends_on:
      - elasticsearch

  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"

  # Optional: Zipkin dependencies job (aggregates service topology)
  zipkin-dependencies:
    image: openzipkin/zipkin-dependencies:3
    environment:
      - STORAGE_TYPE=elasticsearch
      - ES_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    # Run as a cron job daily
    entrypoint: crond -f
```

### Jaeger vs. Zipkin Comparison

| Feature | Jaeger | Zipkin |
|---|---|---|
| **CNCF Status** | Graduated | Not in CNCF |
| **Default storage** | Badger (dev), Cassandra/ES (prod) | In-memory (dev), MySQL/ES/Cassandra (prod) |
| **UI** | Rich, includes service dependency graph, trace comparison | Simpler; focused on trace search and timeline |
| **Sampling** | Head-based (client) + adaptive (collector) | Head-based only |
| **Protocol support** | OTLP, Thrift, Jaeger native | Zipkin B3 (native), OTLP via collector |
| **Kubernetes operator** | ✅ Official Jaeger Operator | ❌ No official operator |
| **OpenTelemetry native** | ✅ OTLP first-class | Partial (via OTLP → Zipkin bridge) |
| **Scalability** | High; designed for large-scale deployment | Moderate; simpler architecture |
| **B3 propagation support** | ✅ Yes (via OTel) | ✅ Native |
| **W3C Trace Context** | ✅ Native | Partial (community libraries) |
| **Best for** | New Kubernetes-native deployments | Existing Zipkin ecosystems, simpler setups |

---

## Sampling Strategies

### Head-Based Sampling

The sampling decision is made **at the start of the trace** (at the root span), before any work is done. All downstream services respect the parent's sampling decision.

```
  Incoming request → Root Span created → Sampling decision: SAMPLE or DROP
                                                │
                     ┌──────────────────────────┤
                     │                          │
               SAMPLE (keep)             DROP (discard)
                     │                          │
             Propagate decision          Pass traceID but
             in headers to all           flag as "not sampled"
             downstream services         downstream also drops
```

**Pros:**
- Simple to implement and reason about
- Low overhead — decision is made once per trace
- Downstream services can skip tracing work if not sampled

**Cons:**
- Cannot sample based on outcome (you don't know yet if the request will error or be slow)
- Errors and slow requests are discarded at the same rate as normal requests

### Tail-Based Sampling

The sampling decision is made **after the trace is complete**, based on the actual outcome (e.g., always keep errors and slow traces).

```
  All spans are collected temporarily in a buffer (Collector or OTel Collector):

  ┌────────────────────────────────────────────────────────────┐
  │                  Tail-Based Sampler Buffer                 │
  │                                                            │
  │  Trace AABB: 200ms, status OK   → Check policy → DROP     │
  │  Trace CCDD: 3500ms, status ERR → Check policy → KEEP     │
  │  Trace EEFF: 150ms, status OK   → Check policy → DROP     │
  │  Trace GGHH: 5000ms, status OK  → Check policy → KEEP     │
  └────────────────────────────────────────────────────────────┘
                         │                │
                    DROP spans        WRITE to storage
```

**Pros:**
- Can always keep 100% of errors and slow traces
- Better signal-to-noise ratio in storage

**Cons:**
- Requires buffering all spans until trace is complete
- Complex to operate (buffer must handle trace completion timeout)
- Higher resource usage in the collector

### Adaptive Sampling

The sampler **automatically adjusts the sampling rate** to hit a target number of traces per second or per operation, regardless of traffic volume.

**Pros:**
- Consistent trace volume even with traffic spikes
- Good balance between cost and visibility

**Cons:**
- More complex to configure and debug
- May undersample during sudden traffic spikes before rate adjusts

### Sampling Strategy Comparison

| Strategy | Decision Point | Keeps Errors | Keeps Slow Requests | Complexity | Storage Cost |
|---|---|---|---|---|---|
| **Constant (always)** | Head | ✅ Yes | ✅ Yes | Very Low | Very High |
| **Probabilistic** | Head | At sample rate | At sample rate | Low | Predictable |
| **Rate Limiting** | Head | At rate limit | At rate limit | Low | Controlled |
| **Adaptive** | Head (dynamic rate) | At rate limit | At rate limit | Medium | Controlled |
| **Tail-Based** | Tail (after completion) | ✅ Always | ✅ Always | High | Low (best) |

### Rate Limiting Configuration (OpenTelemetry Collector)

```yaml
# otelcol-config.yaml — tail-based sampling in the OpenTelemetry Collector
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  # Tail-based sampling — decision made after all spans arrive
  tail_sampling:
    decision_wait: 10s          # Wait 10s for all spans before deciding
    num_traces: 100000          # Max traces in memory buffer
    expected_new_traces_per_sec: 1000

    policies:
      # Always sample errors
      - name: errors-policy
        type: status_code
        status_code: {status_codes: [ERROR]}

      # Always sample slow traces (> 2 seconds)
      - name: slow-traces-policy
        type: latency
        latency: {threshold_ms: 2000}

      # Sample 10% of healthy fast traces
      - name: base-rate-policy
        type: probabilistic
        probabilistic: {sampling_percentage: 10}

exporters:
  otlp:
    endpoint: jaeger-collector:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [tail_sampling]
      exporters: [otlp]
```

---

## Instrumentation Examples

### Manual Instrumentation (Go with OpenTelemetry)

```go
// order_service.go — manual OpenTelemetry instrumentation
package order

import (
    "context"
    "fmt"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("order-service")

type OrderService struct {
    db      Database
    payment PaymentService
}

func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    // Start a span for the entire CreateOrder operation
    ctx, span := tracer.Start(ctx, "CreateOrder",
        trace.WithSpanKind(trace.SpanKindServer),
        trace.WithAttributes(
            attribute.String("user.id", req.UserID),
            attribute.Int("item.count", len(req.Items)),
        ),
    )
    defer span.End()

    // Add an event (log within the span)
    span.AddEvent("Validating order request")

    if err := s.validateOrder(req); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "Order validation failed")
        return nil, fmt.Errorf("validation: %w", err)
    }

    // Child span for database write
    ctx, dbSpan := tracer.Start(ctx, "db.InsertOrder",
        trace.WithSpanKind(trace.SpanKindClient),
        trace.WithAttributes(
            attribute.String("db.system", "postgresql"),
            attribute.String("db.operation", "INSERT"),
            attribute.String("db.name", "orders"),
        ),
    )
    order, err := s.db.InsertOrder(ctx, req)
    dbSpan.End()

    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "Database insert failed")
        return nil, fmt.Errorf("db insert: %w", err)
    }

    // Set the order ID as a span attribute now that we have it
    span.SetAttributes(attribute.String("order.id", order.ID))

    // Child span for payment service call
    order, err = s.processPaymentWithSpan(ctx, order)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "Payment failed")
        return nil, err
    }

    span.SetStatus(codes.Ok, "")
    return order, nil
}

func (s *OrderService) processPaymentWithSpan(ctx context.Context, order *Order) (*Order, error) {
    ctx, span := tracer.Start(ctx, "payment.Process",
        trace.WithSpanKind(trace.SpanKindClient),
        trace.WithAttributes(
            attribute.String("order.id", order.ID),
            attribute.Float64("order.total", order.Total),
        ),
    )
    defer span.End()

    result, err := s.payment.Process(ctx, order)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, err
    }

    span.SetAttributes(attribute.String("payment.id", result.PaymentID))
    return result.Order, nil
}
```

### Auto-Instrumentation (Java Agent)

Java auto-instrumentation uses a Java agent that instruments popular libraries (Spring, JDBC, Kafka, etc.) automatically — zero code changes required.

```bash
# Download the OpenTelemetry Java agent
curl -L https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar \
  -o opentelemetry-javaagent.jar

# Run your Spring Boot app with the agent
java \
  -javaagent:./opentelemetry-javaagent.jar \
  -Dotel.service.name=order-service \
  -Dotel.exporter.otlp.endpoint=http://jaeger:4317 \
  -Dotel.exporter.otlp.protocol=grpc \
  -Dotel.traces.sampler=parentbased_traceidratio \
  -Dotel.traces.sampler.arg=0.1 \
  -Dotel.resource.attributes=deployment.environment=production,service.version=1.2.3 \
  -jar order-service.jar

# What the agent instruments automatically:
# ✅ Spring MVC / Spring Boot HTTP endpoints
# ✅ JDBC / Hibernate / JPA queries
# ✅ Kafka producer/consumer
# ✅ gRPC client/server
# ✅ Redis (Lettuce, Jedis)
# ✅ HTTP clients (OkHttp, HttpClient, RestTemplate, WebClient)
# ✅ Messaging (RabbitMQ, ActiveMQ)
```

### Node.js Auto-Instrumentation

```javascript
// tracing.js — must be loaded BEFORE any application code
'use strict';

const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { ParentBasedSampler, TraceIdRatioBasedSampler } = require('@opentelemetry/sdk-trace-base');

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'order-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: '1.2.3',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: 'production',
  }),

  traceExporter: new OTLPTraceExporter({
    url: 'http://jaeger:4317',
  }),

  sampler: new ParentBasedSampler({
    root: new TraceIdRatioBasedSampler(0.1),  // 10% sampling for new traces
  }),

  // Auto-instruments: http, express, grpc, pg, mysql, redis, kafka, etc.
  instrumentations: [getNodeAutoInstrumentations({
    '@opentelemetry/instrumentation-fs': { enabled: false },  // Disable noisy fs instrumentation
  })],
});

sdk.start();

// Graceful shutdown
process.on('SIGTERM', () => {
  sdk.shutdown().then(() => process.exit(0));
});
```

```bash
# package.json scripts
{
  "scripts": {
    "start": "node -r ./tracing.js app.js"
  },
  "dependencies": {
    "@opentelemetry/sdk-node": "^0.46.0",
    "@opentelemetry/auto-instrumentations-node": "^0.40.0",
    "@opentelemetry/exporter-trace-otlp-grpc": "^0.46.0",
    "@opentelemetry/resources": "^1.19.0",
    "@opentelemetry/semantic-conventions": "^1.19.0"
  }
}
```

---

## Context Propagation

### Over HTTP (Headers)

```go
// Go — HTTP middleware that extracts context from incoming requests
func TracingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Extract W3C traceparent/tracestate headers from incoming request
        ctx := otel.GetTextMapPropagator().Extract(
            r.Context(),
            propagation.HeaderCarrier(r.Header),
        )

        // Start a server-side span as a child of the incoming span
        ctx, span := tracer.Start(ctx, r.URL.Path,
            trace.WithSpanKind(trace.SpanKindServer),
            trace.WithAttributes(
                attribute.String("http.method", r.Method),
                attribute.String("http.url", r.URL.String()),
            ),
        )
        defer span.End()

        // Pass the enriched context to the handler
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Outgoing HTTP call — inject context into headers
func callService(ctx context.Context, url string) {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)

    // This injects:
    // traceparent: 00-<traceID>-<currentSpanID>-01
    otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))

    http.DefaultClient.Do(req)
}
```

### Over gRPC (Metadata)

```go
// Go — gRPC interceptors for trace propagation
import (
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
    "google.golang.org/grpc"
)

// Server-side: extract trace context from incoming gRPC metadata
server := grpc.NewServer(
    grpc.UnaryInterceptor(otelgrpc.UnaryServerInterceptor()),
    grpc.StreamInterceptor(otelgrpc.StreamServerInterceptor()),
)

// Client-side: inject trace context into outgoing gRPC metadata
conn, err := grpc.Dial("payment-service:50051",
    grpc.WithUnaryInterceptor(otelgrpc.UnaryClientInterceptor()),
    grpc.WithStreamInterceptor(otelgrpc.StreamClientInterceptor()),
)

// The interceptors automatically inject/extract these gRPC metadata keys:
// traceparent, tracestate, baggage
```

### Over Message Queues (Kafka Headers)

```go
// Go — Kafka producer with trace context injection
import (
    "github.com/segmentio/kafka-go"
    "go.opentelemetry.io/otel/propagation"
)

type KafkaHeaderCarrier []kafka.Header

func (k *KafkaHeaderCarrier) Get(key string) string {
    for _, h := range *k {
        if h.Key == key {
            return string(h.Value)
        }
    }
    return ""
}

func (k *KafkaHeaderCarrier) Set(key string, value string) {
    *k = append(*k, kafka.Header{Key: key, Value: []byte(value)})
}

func (k *KafkaHeaderCarrier) Keys() []string {
    keys := make([]string, len(*k))
    for i, h := range *k {
        keys[i] = h.Key
    }
    return keys
}

// Producer: inject trace context into Kafka message headers
func produceMessage(ctx context.Context, writer *kafka.Writer, payload []byte) error {
    var headers KafkaHeaderCarrier

    // Inject W3C traceparent into Kafka headers
    otel.GetTextMapPropagator().Inject(ctx, &headers)

    return writer.WriteMessages(ctx, kafka.Message{
        Value:   payload,
        Headers: []kafka.Header(headers),
    })
}

// Consumer: extract trace context from Kafka message headers
func consumeMessage(msg kafka.Message) {
    carrier := KafkaHeaderCarrier(msg.Headers)

    // Extract trace context and create a new span linked to the producer's trace
    ctx := otel.GetTextMapPropagator().Extract(context.Background(), &carrier)
    ctx, span := tracer.Start(ctx, "kafka.consume",
        trace.WithSpanKind(trace.SpanKindConsumer),
        trace.WithAttributes(
            attribute.String("messaging.system", "kafka"),
            attribute.String("messaging.destination", msg.Topic),
            attribute.Int64("messaging.kafka.partition", int64(msg.Partition)),
            attribute.Int64("messaging.kafka.offset", msg.Offset),
        ),
    )
    defer span.End()

    processMessage(ctx, msg.Value)
}
```

---

## Trace Analysis and Finding Bottlenecks

### What to Look For

```
  In the Jaeger/Zipkin trace timeline, look for:

  1. Gaps (sequential waits):
     ┌────────┐         ┌────────┐
     │ SpanA  │         │ SpanB  │
     └────────┘         └────────┘
              ◄────────►
              Gap = queuing, network, or serialization overhead

  2. Long spans (direct bottlenecks):
     ┌─────────────────────────────────────────────────────┐
     │                    DB Query (3500ms)                │  ← BOTTLENECK
     └─────────────────────────────────────────────────────┘

  3. Sequential vs. parallel:
     Sequential (slow):        Parallel (fast):
     ┌──────┐                  ┌──────┐┌──────┐
     │ SvcA │                  │ SvcA ││ SvcB │
     └──────┘                  └──────┘└──────┘
             ┌──────┐
             │ SvcB │          Both complete in max(A,B) time
             └──────┘

  4. Retries (span count >> expected):
     Span: "payment.charge"  ×6 identical spans → retry storm

  5. N+1 queries (many identical short spans):
     db.query ×100 (each 5ms = 500ms total) → needs batch query
```

### Critical Path Analysis

The **critical path** is the longest chain of causally-linked spans — the sequence that determined the total trace duration. Optimizing any span NOT on the critical path has no effect on total latency.

```
  Total trace: 500ms

  Path 1 (critical): SpanA(20ms) → SpanC(350ms) → SpanE(80ms) = 450ms ← Critical path
  Path 2:            SpanA(20ms) → SpanD(200ms)                = 220ms

  To reduce total latency:
  ✅ Optimize SpanC (on critical path) → reduces total duration
  ❌ Optimize SpanD (not on critical path) → no improvement to user
```

### Flame Graph Description

A **flame graph** represents trace data as a stacked bar chart:
- **X-axis**: time (left = start, right = end)
- **Y-axis**: span depth (bottom = root, top = leaves)
- **Width of bar**: proportional to span duration
- **Color**: usually by service or span status

```
  Flame Graph Layout:

  ▲ Span depth
  │
  4│        [db.query(80ms)]
  3│    [fraud.check(120ms)       ]
  2│  [payment.process(160ms)          ]
  1│[order.create(200ms)                    ][inventory.check(40ms)]
  0│[HTTP POST /checkout(250ms)                                        ]
   └────────────────────────────────────────────────────────────────────► Time
    0ms      50ms     100ms    150ms    200ms    250ms
```

### Useful Attributes to Add to Spans

| Attribute | Example Value | Why It Helps |
|---|---|---|
| `http.method` | `"POST"` | Filter by HTTP verb in Jaeger search |
| `http.status_code` | `200` | Quickly spot errors without reading span status |
| `db.statement` | `"SELECT * FROM orders WHERE ..."` | Find slow queries (redact sensitive values) |
| `db.rows_affected` | `1` | Detect unexpected zero-row updates |
| `user.id` | `"usr_abc123"` | Find all traces for a specific user for support tickets |
| `order.id` | `"ord_xyz789"` | Correlate all spans for a specific business entity |
| `retry.count` | `2` | Spot retry storms |
| `cache.hit` | `false` | Understand cache miss rates |
| `queue.depth` | `1542` | Track queue backpressure |
| `feature_flag.name` | `"new_checkout_flow"` | Correlate A/B test variants with performance |

---

## Next Steps

| File | Topic |
|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Observability Overview — three pillars and golden signals |
| [01-METRICS.md](01-METRICS.md) | Metrics — Prometheus, PromQL, and Grafana |
| [02-LOGGING.md](02-LOGGING.md) | Logging — structured logs, ELK stack, Loki |
| [04-OPENTELEMETRY.md](04-OPENTELEMETRY.md) | OpenTelemetry — unified SDK, Collector, auto-instrumentation |
| [05-ALERTING.md](05-ALERTING.md) | Alerting — Alertmanager, alert rules, incident routing |
| [06-SLOS-AND-SLIS.md](06-SLOS-AND-SLIS.md) | SLOs and SLIs — error budgets and burn rate alerts |
| [08-INCIDENT-RESPONSE.md](08-INCIDENT-RESPONSE.md) | Incident Response — using traces during incidents |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path — structured guide by role |
