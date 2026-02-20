# Observability Best Practices

A comprehensive guide to building, operating, and scaling observable systems — covering the Four Golden Signals, RED/USE methods, instrumentation standards, cardinality management, health checks, and production-ready patterns.

---

## Table of Contents

- [The Four Golden Signals](#the-four-golden-signals)
  - [Latency](#latency)
  - [Traffic](#traffic)
  - [Errors](#errors)
  - [Saturation](#saturation)
  - [Golden Signals Summary Table](#golden-signals-summary-table)
- [RED Method](#red-method)
- [USE Method](#use-method)
- [Instrumentation Standards](#instrumentation-standards)
  - [Metric Naming Conventions](#metric-naming-conventions)
  - [Label Guidelines](#label-guidelines)
  - [Unit Suffix Convention](#unit-suffix-convention)
  - [Histogram Bucket Selection](#histogram-bucket-selection)
- [Cardinality Management](#cardinality-management)
- [Correlation IDs and Request Tracing](#correlation-ids-and-request-tracing)
- [Sampling Strategies](#sampling-strategies)
- [Cost Optimization](#cost-optimization)
- [Health Check Patterns](#health-check-patterns)
  - [Liveness Probes](#liveness-probes)
  - [Readiness Probes](#readiness-probes)
  - [Startup Probes](#startup-probes)
  - [Deep Health Checks](#deep-health-checks)
- [Dashboard Design Principles](#dashboard-design-principles)
- [Observability as Code](#observability-as-code)
- [Runbook Automation](#runbook-automation)
- [Next Steps](#next-steps)
- [Version History](#version-history)

---

## The Four Golden Signals

The Four Golden Signals originated in the **Google Site Reliability Engineering (SRE) book** (2016), where Google described how its SRE teams focus on just four metrics to understand the health of any service. This framework emerged from years of operating internet-scale systems and recognising that the hundreds of metrics produced by modern infrastructure can be distilled to four signals that directly reflect **user experience**.

> "If you can only measure four metrics of your user-facing system, focus on these four." — Google SRE Book, Chapter 6

The power of this framework is its simplicity: rather than alert on infrastructure internals, you alert on what users actually experience.

```
         USER EXPERIENCE
               │
    ┌──────────┼──────────┐
    │          │          │
    ▼          ▼          ▼
 Latency    Errors    Traffic
               │
            Saturation
         (leading indicator)
```

### Latency

**Definition:** The time it takes to service a request. Critically, this includes both **successful** and **failed** requests — a fast error response is still an error and should be tracked separately.

**Why percentiles, not averages:**

Averages hide the distribution of latency. If 95% of requests complete in 10ms but 5% take 10 seconds, the average might look acceptable while the tail experience is terrible.

```
Request Times (ms): [5, 8, 7, 10, 9, 6, 8, 7, 5, 10000]
Average:  1006ms   ← looks terrible
p50:        8ms    ← typical user experience
p95:       10ms    ← slightly slower users
p99:    10000ms    ← the one user who waited 10 seconds
```

**PromQL Examples:**

```promql
# p50 latency for the HTTP handler
histogram_quantile(0.50,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, handler)
)

# p95 latency — alerts here to catch degradation early
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, handler)
)

# p99 latency — SLO boundary, tail latency
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, handler)
)

# Latency across all handlers (fleet-wide)
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)
```

**Alert Recommendation:** Alert when p99 latency exceeds your SLO threshold (e.g., > 1 second for user-facing APIs) or when p95 latency more than doubles from baseline.

---

### Traffic

**Definition:** A measure of how much demand is being placed on your system. For web services this is requests per second. For other systems it may be messages consumed per second, queries per second, or bytes transferred.

**Why it matters:** Traffic context is essential for interpreting all other signals. A 5% error rate during 10 RPS is different from a 5% error rate during 10,000 RPS. Traffic also helps you plan capacity.

**PromQL Examples:**

```promql
# Total request rate across all handlers
sum(rate(http_requests_total[5m]))

# Request rate per handler
sum(rate(http_requests_total[5m])) by (handler)

# Request rate per status code class
sum(rate(http_requests_total[5m])) by (status_code)

# Requests per minute (useful for dashboards)
sum(rate(http_requests_total[1m])) * 60
```

**Alert Recommendation:** Alert on sudden traffic drops (may indicate upstream failures) or traffic spikes (may indicate attacks or bugs). Use percentage changes: `rate_now / rate_1h_ago > 3` (3× traffic spike).

---

### Errors

**Definition:** The rate of requests that fail — either explicitly (HTTP 5xx, gRPC status != OK) or implicitly (HTTP 200 with an error body, or incorrect results).

**4xx vs 5xx distinction:**

| Status Class | Meaning | Alert? |
|---|---|---|
| 2xx | Success | No (unless missing) |
| 3xx | Redirect | Usually no |
| 4xx | Client error (bad request, auth failure) | Sometimes (spike may indicate attack or breaking API change) |
| 5xx | Server error (our fault) | Yes — this is your error rate SLO |

> **Important:** Do not include 4xx in your primary error rate SLO. A `401 Unauthorized` is the correct response to an unauthenticated request — it is not a service failure.

**PromQL Examples:**

```promql
# 5xx error rate (server-side errors only)
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
  /
sum(rate(http_requests_total[5m]))

# 4xx rate (client errors, separate tracking)
sum(rate(http_requests_total{status_code=~"4.."}[5m]))
  /
sum(rate(http_requests_total[5m]))

# Percentage of successful requests (for SLO dashboards)
(
  sum(rate(http_requests_total{status_code=~"2.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
) * 100

# Alert: error rate > 1% for 5 minutes
- alert: HighErrorRate
  expr: |
    sum(rate(http_requests_total{status_code=~"5.."}[5m]))
    /
    sum(rate(http_requests_total[5m])) > 0.01
  for: 5m
```

---

### Saturation

**Definition:** How "full" your service is. A measure of the constrained resource — the thing most likely to limit throughput. For CPU-bound services it is CPU utilisation; for memory-bound it is memory; for I/O-bound it is disk or network. **Saturation is a leading indicator** — it predicts latency and error degradation before users feel it.

**PromQL Examples:**

```promql
# CPU utilisation per pod (%)
100 - (
  avg by (pod) (
    rate(container_cpu_usage_seconds_total{container!=""}[5m])
  )
  /
  avg by (pod) (
    kube_pod_container_resource_limits{resource="cpu"}
  )
) * 100

# Memory utilisation per pod (%)
(
  container_memory_working_set_bytes{container!=""}
  /
  kube_pod_container_resource_limits{resource="memory"}
) * 100

# Disk utilisation per node
(
  1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)
) * 100

# Thread pool saturation (custom metric)
# Ratio of active threads to max pool size
http_server_active_threads / http_server_max_threads

# Alert: memory saturation > 85%
- alert: HighMemorySaturation
  expr: |
    container_memory_working_set_bytes
    /
    kube_pod_container_resource_limits{resource="memory"} > 0.85
  for: 10m
```

**Alert Recommendation:** Alert when saturation exceeds 80% sustained — this gives you time to react before the service degrades.

---

### Golden Signals Summary Table

| Signal | What It Measures | PromQL Example | Alert Threshold |
|--------|-----------------|----------------|-----------------|
| **Latency** | Time to serve a request (p50/p95/p99) | `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))` | p99 > SLO target (e.g., 1s) |
| **Traffic** | Demand on the system (RPS, QPS) | `sum(rate(http_requests_total[5m]))` | > 3× baseline (spike) or < 20% (drop) |
| **Errors** | Rate of failed requests (5xx) | `sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))` | > 1% for 5 minutes |
| **Saturation** | Resource utilisation (CPU, memory, disk) | `container_memory_working_set_bytes / kube_pod_container_resource_limits{resource="memory"}` | > 80% sustained for 10 minutes |

---

## RED Method

The RED method was coined by **Tom Wilkie** at Weaveworks and is particularly well-suited for **microservices and request-driven systems**. RED focuses entirely on the service's API surface from the perspective of an individual request.

| Letter | Stands For | Definition |
|--------|------------|-----------|
| **R** | Rate | Requests per second |
| **E** | Errors | Number of failing requests per second |
| **D** | Duration | Distribution of request durations (latency) |

### RED vs Golden Signals Relationship

RED is essentially a subset of the Four Golden Signals tailored for services:

```
Four Golden Signals       RED Method
─────────────────        ─────────────
Latency           ──▶    Duration
Traffic           ──▶    Rate
Errors            ──▶    Errors
Saturation        ──▶    (not covered — use USE for this)
```

### RED Metric Table

| Metric | Definition | PromQL Example |
|--------|-----------|----------------|
| Rate | Requests per second handled by the service | `sum(rate(http_requests_total[5m])) by (service)` |
| Errors | Failed requests per second | `sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)` |
| Duration (p50) | Median response time | `histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))` |
| Duration (p99) | 99th percentile response time | `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))` |
| Error % | Percentage of requests that fail | `(sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))) * 100` |

### When to Use RED vs Golden Signals vs USE

| Framework | Best For | Focus | Not Good For |
|-----------|---------|-------|--------------|
| **Four Golden Signals** | Any user-facing service | User experience | Infrastructure internals |
| **RED** | Microservices, APIs, request-driven systems | Per-request latency/error/rate | Infrastructure resources |
| **USE** | Infrastructure components, hosts, databases | Resource utilisation | High-level user experience |

**Rule of thumb:**
- **RED** for application-level services (what the user sees)
- **USE** for infrastructure (what supports the application)
- **Golden Signals** for overall SLO/SLA reporting

---

## USE Method

The USE method was created by **Brendan Gregg** and is designed for diagnosing performance issues in **infrastructure resources** — CPUs, memory, disks, network interfaces, storage controllers.

| Letter | Stands For | Definition |
|--------|------------|-----------|
| **U** | Utilization | Average time the resource was busy serving work |
| **S** | Saturation | Amount of work queued that cannot be serviced (backpressure) |
| **E** | Errors | Count of error events |

For each resource type, ask three questions:
1. What is the utilisation? (Are we using most of the capacity?)
2. What is the saturation? (Is work queuing up?)
3. Are there errors? (Are operations failing?)

### USE Method Resource Table

| Resource | Utilisation Query | Saturation Query | Errors Query |
|----------|-----------------|-----------------|-------------|
| **CPU** | `avg(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance)` | `avg(node_load1) by (instance) / count(node_cpu_seconds_total{mode="idle"}) by (instance)` | `rate(node_context_switches_total[5m])` (excessive = saturation indicator) |
| **Memory** | `1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)` | `rate(node_vmstat_pgmajfault[5m])` (major page faults = swapping) | `node_memory_HardwareCorrupted_bytes > 0` |
| **Disk** | `rate(node_disk_io_time_seconds_total[5m])` | `rate(node_disk_io_time_weighted_seconds_total[5m])` (I/O wait time) | `rate(node_disk_read_errors_total[5m]) + rate(node_disk_write_errors_total[5m])` |
| **Network** | `rate(node_network_transmit_bytes_total[5m]) / node_network_speed_bytes` | `rate(node_network_transmit_drop_total[5m])` (dropped packets) | `rate(node_network_transmit_errs_total[5m]) + rate(node_network_receive_errs_total[5m])` |

### Combining USE with RED

In practice, you need both. Use the following workflow when investigating an incident:

```
Symptom (RED alert fires)
│
├── Error rate high?
│     └── Check application logs, trace errors → fix code/config
│
├── Latency high (Duration)?
│     ├── Is traffic (Rate) normal?
│     │     └── If yes: investigate saturation (USE)
│     └── CPU/Memory/Disk saturated? (USE)
│           └── Scale up, optimize, or shed load
│
└── Rate dropped?
      └── Check upstream, check error rate, check saturation
```

---

## Instrumentation Standards

### Metric Naming Conventions

Consistent naming conventions make metrics discoverable, self-documenting, and composable. Follow the Prometheus community conventions.

**Format:** `<namespace>_<subsystem>_<name>_<unit>_total` (for counters)

| Rule | Good Example | Bad Example | Why |
|------|-------------|-------------|-----|
| Use snake_case | `http_requests_total` | `httpRequestsTotal` | Prometheus convention |
| Include namespace | `myapp_http_requests_total` | `http_requests_total` | Avoid conflicts across services |
| Include unit suffix | `request_duration_seconds` | `request_duration` | Makes unit unambiguous |
| Counters end in `_total` | `errors_total` | `errors` | Prometheus naming standard |
| No verb in counter names | `http_requests_total` | `http_requests_counted_total` | Concise |
| Base units, not display units | `_seconds` | `_milliseconds` | Convert at query time |
| `_bytes` not `_kilobytes` | `payload_size_bytes` | `payload_size_kb` | Base SI unit |
| Histogram has `_bucket`, `_sum`, `_count` | (automatic with Prometheus SDK) | Custom histogram without these | Standard aggregation |
| `_ratio` for ratios (0–1) | `cache_hit_ratio` | `cache_hit_percent` | Percentages are confusing |
| Boolean status as gauge 0/1 | `service_up` (1=up, 0=down) | `service_status_string` | Queryable in PromQL |
| Avoid redundant prefixes | `app_requests_total` | `app_app_requests_total` | Namespace + subsystem clarity |
| Consistent tense | `_created` (timestamp) | `_create_time` | Standard pattern |

### Label Guidelines

Labels are powerful for slicing and dicing metrics, but they come with a cost: each unique label combination creates a new **time series**. Poor label design leads to cardinality explosions.

**What makes a good label:**

```
Good label properties:
  ✓ Low cardinality (< 100 unique values)
  ✓ Describes a dimension for aggregation
  ✓ Stable values (not changing per request)
  ✓ Useful for filtering or grouping

Bad label properties:
  ✗ High cardinality (millions of unique values)
  ✗ Contains user-specific or request-specific data
  ✗ Dynamic values that grow unboundedly
  ✗ Redundant with the metric name itself
```

| Label | Type | Cardinality | Verdict |
|-------|------|-------------|---------|
| `method` (GET/POST/PUT/DELETE) | Dimension | 4–7 | ✅ Good |
| `status_code` (200/404/500) | Dimension | ~10 | ✅ Good |
| `service` (order-service, payment) | Dimension | ~20 | ✅ Good |
| `region` (us-east-1, eu-west-1) | Dimension | ~10 | ✅ Good |
| `version` (v1.2.3) | Dimension | ~20 versions | ✅ Good (with cleanup) |
| `user_id` | Per-user | Millions | ❌ Never |
| `request_id` | Per-request | Billions | ❌ Never |
| `session_id` | Per-session | Millions | ❌ Never |
| `url_path` (raw, not template) | Unbounded | Billions | ❌ Never |
| `query_string` | Unbounded | Billions | ❌ Never |
| `customer_name` | Unbounded | Thousands | ❌ Avoid |

**Use labels to answer operational questions, not business intelligence questions.** For user-level analysis, use traces with exemplars.

### Unit Suffix Convention

| Metric Type | Correct Suffix | Example |
|-------------|---------------|---------|
| Time / Duration | `_seconds` | `http_request_duration_seconds` |
| File/memory size | `_bytes` | `process_resident_memory_bytes` |
| Temperatures | `_celsius` | `hardware_temperature_celsius` |
| Electrical | `_amperes`, `_volts` | `psu_current_amperes` |
| Ratios (0–1) | `_ratio` | `cache_hit_ratio` |
| Totals (counters) | `_total` | `http_requests_total` |
| Queue depth | `_queue_depth` | `worker_queue_depth` |
| Percentages | Avoid — use `_ratio` | `cpu_utilisation_ratio` |
| Operations per second | Let PromQL compute with `rate()` | `database_queries_total` |
| In-progress items | `_inprogress` | `http_requests_inprogress` |

### Histogram Bucket Selection

Buckets define the resolution of your latency distribution. Choosing appropriate buckets avoids either too-coarse or too-fine granularity.

**Rules for bucket selection:**

1. Cover the entire expected range including outliers (up to 10× your SLO)
2. Have more resolution around your SLO targets (e.g., more buckets near 100ms–1s)
3. Use exponential progression (each bucket ~2× the previous)
4. Include `+Inf` (always automatic in Prometheus SDKs)

```go
// Go example: bucket configuration for a web API with 200ms SLO
prometheus.NewHistogramVec(prometheus.HistogramOpts{
    Name: "http_request_duration_seconds",
    Help: "Duration of HTTP requests in seconds",
    Buckets: []float64{
        0.005, // 5ms
        0.010, // 10ms
        0.025, // 25ms
        0.050, // 50ms
        0.100, // 100ms  ← fine resolution near SLO
        0.150, // 150ms
        0.200, // 200ms  ← SLO target
        0.300, // 300ms
        0.500, // 500ms
        1.000, // 1s
        2.500, // 2.5s   ← degraded service
        5.000, // 5s
        10.00, // 10s    ← circuit breaker territory
    },
}, []string{"method", "handler", "status_code"})
```

**Default Prometheus SDK buckets are usually wrong** (they cover 5ms to 10s uniformly), so always configure custom buckets appropriate to your service's latency profile.

---

## Cardinality Management

### What Is Cardinality and Why It Matters

**Time series cardinality** is the number of unique label-value combinations for a given metric. Each unique combination is a separate time series stored in Prometheus's TSDB.

```
http_requests_total{method="GET", status="200", handler="/api/users"}   ← 1 series
http_requests_total{method="GET", status="404", handler="/api/users"}   ← 1 series
http_requests_total{method="POST", status="201", handler="/api/orders"} ← 1 series
...
```

Total series = (# of method values) × (# of status values) × (# of handler values)

### How Label Explosion Grows Time Series Count

```
Metric: http_requests_total

With 3 labels (safe):
  method=3 × status=10 × handler=20 = 600 series

Add service label (still safe):
  method=3 × status=10 × handler=20 × service=15 = 9,000 series

Add region label (caution):
  ... × region=10 = 90,000 series

Add user_id label (DANGER):
  ... × user_id=1,000,000 = 90,000,000,000 series ← system OOM

          600 → 9,000 → 90,000 → 90B series
            │       │        │         │
            └───────┴────────┴─────────┘
            Each label multiplies the total
```

### High Cardinality Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Using `user_id` as a label | Millions of series, memory exhaustion | Use traces/exemplars for per-user data |
| Using raw `url_path` as label | Unbounded growth (e.g., `/api/users/123`) | Normalise to template: `/api/users/{id}` |
| Using `request_id` as label | Every request = new series | Log request ID, link via exemplar |
| Using `session_id` as label | Unbounded per-session series | Traces with session context |
| Using `version` without cleanup | Series accumulate on each deploy | Set retention or use `max_stale_age` |
| Including environment in every metric | Doubles series for staging/prod | Use separate Prometheus instances |
| Using enum with growing values | Open-ended business enums | Cap cardinality, aggregate others |
| Logs with metrics mixed cardinality | Log query creates metric label pollution | Keep metrics and logs separate |

### Prometheus Cardinality Tools

**1. TSDB Status page** — built-in UI at `http://prometheus:9090/tsdb-status`

```bash
# Shows top 10 metrics by series count
curl http://localhost:9090/api/v1/status/tsdb | jq '.data.seriesCountByMetricName[:10]'
```

**2. `promtool tsdb analyze`** — analyzes a local TSDB block

```bash
# Analyze TSDB block for cardinality
promtool tsdb analyze /var/lib/prometheus/data

# Output shows:
# - highest cardinality metrics
# - highest cardinality label names
# - correlation between labels
```

**3. Prometheus UI Explore** — query `count({__name__=~".+"}) by (__name__)` to see series per metric

```promql
# Top 10 metrics by time series count
topk(10, count by (__name__)({__name__=~".+"}))

# Count total active time series
count({__name__=~".+"})
```

### Cardinality Budget Guidelines

| Total Active Series | Classification | Action |
|---|---|---|
| < 100,000 | Healthy | No action needed |
| 100,000 – 500,000 | Monitor | Review new metrics before adding labels |
| 500,000 – 1,000,000 | Warning | Audit top cardinality metrics, consider federation |
| 1,000,000 – 5,000,000 | High | Enforce cardinality limits, drop high-card series |
| > 5,000,000 | Critical | Emergency reduction; consider Thanos/Cortex with limits |

---

## Correlation IDs and Request Tracing

### What Are Correlation IDs?

A **correlation ID** is a unique identifier attached to a request at its entry point (API gateway, load balancer, or first service) and **propagated through every downstream service** in every log message, trace span, and event. It enables you to:

- Find all logs related to a single user request across 10+ services
- Link a support ticket to specific trace data
- Correlate metrics spikes with specific request batches

```
Client → API Gateway → Order Service → Payment Service → Notification Service
           │                │                │                   │
           └────── correlation_id: "abc-123" propagated through all ──────┘
                  (in HTTP headers, log fields, and trace context)
```

### HTTP Middleware Implementation — Go

```go
package middleware

import (
    "context"
    "net/http"

    "github.com/google/uuid"
    "go.uber.org/zap"
)

type contextKey string

const CorrelationIDKey contextKey = "correlation_id"

// CorrelationIDMiddleware injects or propagates a correlation ID
func CorrelationIDMiddleware(logger *zap.Logger, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Check if upstream already sent a correlation ID
        correlationID := r.Header.Get("X-Correlation-ID")
        if correlationID == "" {
            // Generate a new one at the edge
            correlationID = uuid.New().String()
        }

        // Add to response headers so clients can reference it
        w.Header().Set("X-Correlation-ID", correlationID)

        // Store in context for use by handlers
        ctx := context.WithValue(r.Context(), CorrelationIDKey, correlationID)

        // Log all requests with the correlation ID
        logger.Info("request started",
            zap.String("correlation_id", correlationID),
            zap.String("method", r.Method),
            zap.String("path", r.URL.Path),
        )

        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// GetCorrelationID retrieves the correlation ID from context
func GetCorrelationID(ctx context.Context) string {
    if id, ok := ctx.Value(CorrelationIDKey).(string); ok {
        return id
    }
    return "unknown"
}

// PropagateCorrelationID adds the correlation ID to an outgoing HTTP request
func PropagateCorrelationID(ctx context.Context, req *http.Request) {
    if id := GetCorrelationID(ctx); id != "" {
        req.Header.Set("X-Correlation-ID", id)
    }
}
```

### HTTP Middleware Implementation — Python / FastAPI

```python
import uuid
from fastapi import FastAPI, Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
import structlog

logger = structlog.get_logger()

class CorrelationIDMiddleware(BaseHTTPMiddleware):
    """Inject or propagate X-Correlation-ID on every request."""

    async def dispatch(self, request: Request, call_next) -> Response:
        # Accept correlation ID from upstream, or generate a new one
        correlation_id = request.headers.get("X-Correlation-ID", str(uuid.uuid4()))

        # Bind to structlog context for all log messages in this request
        structlog.contextvars.bind_contextvars(correlation_id=correlation_id)

        # Store on request state so route handlers can access it
        request.state.correlation_id = correlation_id

        response = await call_next(request)

        # Echo back in response header
        response.headers["X-Correlation-ID"] = correlation_id

        # Clear context to prevent leakage to the next request
        structlog.contextvars.clear_contextvars()

        return response

app = FastAPI()
app.add_middleware(CorrelationIDMiddleware)

@app.get("/orders/{order_id}")
async def get_order(order_id: str, request: Request):
    logger.info("fetching order",
                order_id=order_id,
                correlation_id=request.state.correlation_id)
    # ...
```

### Propagation Through Message Queues

Correlation IDs must survive asynchronous handoffs. Include them in message envelopes:

```json
{
  "message_id": "msg-789",
  "correlation_id": "abc-123",
  "causation_id": "cmd-456",
  "timestamp": "2024-01-15T10:30:00Z",
  "event_type": "OrderPlaced",
  "payload": {
    "order_id": "order-001",
    "customer_id": "cust-999"
  }
}
```

```python
# Publishing with correlation ID (Python + Kafka)
producer.send(
    topic="order-events",
    value=event_payload,
    headers=[
        ("X-Correlation-ID", correlation_id.encode()),
        ("X-Causation-ID", causation_id.encode()),
    ]
)

# Consuming and extracting correlation ID
for message in consumer:
    headers = {k: v.decode() for k, v in message.headers}
    correlation_id = headers.get("X-Correlation-ID", "unknown")
    structlog.contextvars.bind_contextvars(correlation_id=correlation_id)
```

### Adding Correlation ID to All Log Lines

With structured logging, bind the correlation ID once per request and it automatically appears in every log line:

```go
// Go with zap — create a child logger per request
func Handler(w http.ResponseWriter, r *http.Request) {
    correlationID := GetCorrelationID(r.Context())
    log := logger.With(zap.String("correlation_id", correlationID))

    log.Info("processing order")       // includes correlation_id
    result, err := processOrder(r)
    if err != nil {
        log.Error("order failed", zap.Error(err))  // includes correlation_id
    }
    log.Info("order complete")         // includes correlation_id
}
```

### Correlation ID Format Recommendations

| Format | Example | Pros | Cons |
|--------|---------|------|------|
| **UUID v4** | `550e8400-e29b-41d4-a716-446655440000` | Universally unique, no coordination | 36 chars, no embedded info |
| **ULID** | `01ARZ3NDEKTSV4RRFFQ69G5FAV` | Sortable, URL-safe, 26 chars | Less widespread |
| **W3C Trace Context** | `00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01` | OTel native, links to traces | Long, complex format |
| **Short UUID** | `abc123de` | Short, URL-safe | Higher collision probability |
| **Snowflake ID** | `1234567890123456789` | Embeds timestamp, sortable | Requires ID generation service |

**Recommendation:** Use **W3C Trace Context (`traceparent`)** if you are using OpenTelemetry — it is both a correlation ID and a distributed trace link. Fall back to UUID v4 if not using distributed tracing.

---

## Sampling Strategies

### When to Sample

Sampling is necessary when trace or log volume is too high to store economically. At 10,000 RPS with 5 spans per request, you generate **50,000 spans/second** = **4.3 billion spans/day**. Full sampling is usually not economical above ~1,000 RPS.

### Head-Based vs Tail-Based Sampling

```
HEAD-BASED SAMPLING
───────────────────
Decision made at trace START (request entry point)
  └── "Sample 1% of all traffic"
  └── Fast, low overhead
  └── Problem: cannot target important traces (errors, slow requests)
  └── You may miss the one slow request in 1000

TAIL-BASED SAMPLING
───────────────────
Decision made at trace END (after all spans collected)
  └── "Sample 100% of traces with errors"
  └── "Sample 100% of traces > 2 seconds"
  └── "Sample 1% of everything else"
  └── Requires buffering spans in collector (cost)
  └── Problem: higher infrastructure cost, more complexity
```

### Representative Sampling Strategies

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| **Probabilistic (Head)** | Sample X% of all traces uniformly | Default for most services |
| **Rate-limiting** | Max N traces/second regardless of traffic | Protect storage under load |
| **Error sampling** | Always sample traces with errors | Tail-based, critical for debugging |
| **Slow trace sampling** | Always sample traces above p99 latency | Tail-based, SLO breach investigation |
| **Per-service** | Different rates per service (10% for high-volume, 100% for low) | Mixed-traffic architectures |
| **Adaptive** | Automatically adjust rate based on error budget consumption | Sophisticated, requires SLO tracking |

### Sampling Configuration Examples

**OpenTelemetry Collector — tail-based sampling:**

```yaml
# otel-collector-config.yaml
processors:
  tail_sampling:
    decision_wait: 10s          # Buffer spans for 10s before deciding
    num_traces: 100000          # Max traces buffered
    expected_new_traces_per_sec: 10000
    policies:
      # Always sample errors
      - name: errors-policy
        type: status_code
        status_code: {status_codes: [ERROR]}

      # Always sample slow requests (> 2s)
      - name: slow-traces-policy
        type: latency
        latency: {threshold_ms: 2000}

      # Sample 2% of everything else
      - name: default-policy
        type: probabilistic
        probabilistic: {sampling_percentage: 2}
```

**Jaeger — probabilistic sampling:**

```yaml
# jaeger-agent configuration
sampling:
  strategies_file: /etc/jaeger/sampling_strategies.json
```

```json
{
  "service_strategies": [
    {
      "service": "order-service",
      "type": "probabilistic",
      "param": 0.1
    },
    {
      "service": "payment-service",
      "type": "probabilistic",
      "param": 1.0
    }
  ],
  "default_strategy": {
    "type": "probabilistic",
    "param": 0.01
  }
}
```

### Sampling Rate Recommendations by Traffic Volume

| Traffic Volume | Recommended Head Sample Rate | Tail-Based Additions |
|---------------|------------------------------|---------------------|
| < 100 RPS | 100% | Not needed |
| 100 – 1,000 RPS | 10–100% | Optional error sampling |
| 1,000 – 10,000 RPS | 1–10% | Error + slow traces at 100% |
| 10,000 – 100,000 RPS | 0.1–1% | Error + slow traces at 100% |
| > 100,000 RPS | 0.01–0.1% | Error + slow + adaptive |

---

## Cost Optimization

### Metric Cardinality

High metric cardinality is the primary driver of Prometheus memory and storage costs.

```yaml
# Drop unnecessary high-cardinality labels with metric_relabel_configs
# in prometheus.yml scrape configuration
scrape_configs:
  - job_name: "my-service"
    static_configs:
      - targets: ["my-service:8080"]
    metric_relabel_configs:
      # Drop go runtime metrics (useful only for library developers)
      - source_labels: [__name__]
        regex: "go_(memstats|gc|goroutines).*"
        action: drop

      # Remove high-cardinality label from specific metric
      - source_labels: [__name__]
        regex: "http_request_duration_seconds.*"
        target_label: url
        replacement: ""
        action: replace

      # Aggregate low-value metrics before storage
      # (use recording rules for this instead)
```

**Use recording rules to pre-aggregate:**

```yaml
# rules/aggregations.yml
groups:
  - name: aggregations
    interval: 1m
    rules:
      # Pre-aggregate request rate by service (drops handler/method granularity)
      - record: job:http_requests_total:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)
```

### Log Volume

| Strategy | Description | Saving |
|----------|------------|--------|
| **Log level discipline** | DEBUG off in production, INFO for business events only | 60–80% reduction |
| **Sampling high-volume logs** | Log 1% of successful requests, 100% of errors | 80–90% reduction on success paths |
| **Request body sampling** | Never log full request/response bodies; log only IDs | Varies |
| **Compression** | Enable gzip/zstd in log shipper | 70–85% storage saving |
| **Short retention** | Keep raw logs 7 days, aggregated 90 days | Linear with retention |
| **Filter before shipping** | Drop health check and metrics scrape logs in agent | 5–15% reduction |

```yaml
# Fluent Bit: drop Kubernetes liveness probe logs before shipping
[FILTER]
    Name    grep
    Match   *
    Exclude log /healthz
    Exclude log /metrics
    Exclude log /readyz
```

### Trace Sampling

Traces are typically the most expensive signal. Apply tail-based sampling to maximise value per byte:

```
Always sample (100%):
  - All traces with errors
  - All traces with status_code = 5xx
  - Traces matching SLO breach criteria (latency > threshold)

Sample proportionally:
  - 1–5% of everything else (head-based)
```

### Cost Comparison Table

| Approach | Cost Impact | Tradeoff |
|----------|------------|----------|
| Sample traces at 1% (head) | 99% trace cost reduction | Miss rare errors |
| Tail-based error sampling | 80-95% trace cost reduction | Miss edge cases |
| Drop go runtime metrics | 10-20% metric cost reduction | Lose Go internals visibility |
| Recording rules aggregation | 30-50% query cost reduction | Lose label granularity |
| Reduce log level to WARN | 70-90% log cost reduction | Miss INFO operational context |
| Async log shipping | 5-10% cost reduction | Potential loss on crash |
| Loki label cardinality limits | 50-70% Loki storage reduction | Fewer indexed label combinations |
| Prometheus federation | Scales cost linearly | Adds infrastructure complexity |

---

## Health Check Patterns

### Liveness Probes

A **liveness probe** answers: "Is the process alive and not stuck in a deadlock?" Kubernetes will **restart** the container if this probe fails.

**What to check:** Only the process itself — not dependencies. A deadlocked process should fail; a temporarily unreachable database should not cause a restart.

```yaml
# kubernetes/deployment.yaml
livenessProbe:
  httpGet:
    path: /livez
    port: 8080
    scheme: HTTP
  initialDelaySeconds: 15   # Wait for app to start
  periodSeconds: 20          # Check every 20s
  failureThreshold: 3        # Restart after 3 failures (60s window)
  timeoutSeconds: 5          # Probe times out after 5s
  successThreshold: 1        # 1 success to mark healthy
```

**What `/livez` should do:**
- Return HTTP 200 if the process event loop / goroutine pool is functioning
- Return HTTP 503 if the process is deadlocked
- Never check external dependencies
- Complete in < 100ms

### Readiness Probes

A **readiness probe** answers: "Is this instance ready to receive traffic?" Kubernetes will **stop routing traffic** to the pod if this probe fails — but will NOT restart the container.

**What to check:** Dependencies that are required to serve requests (database connection pool warm, cache loaded, configuration fetched).

```yaml
readinessProbe:
  httpGet:
    path: /readyz
    port: 8080
  initialDelaySeconds: 5    # Check after 5s (faster than liveness)
  periodSeconds: 10
  failureThreshold: 3       # Remove from LB after 3 failures (30s)
  timeoutSeconds: 5
  successThreshold: 1
```

**What `/readyz` should do:**
- Check that the database connection pool has at least 1 connection
- Check that required caches are populated
- Check that required configuration is loaded
- Return 503 + JSON body explaining what is not ready

### Startup Probes

A **startup probe** handles slow-starting applications (JVM warm-up, model loading, large cache population). It **disables liveness and readiness checks** until the startup probe succeeds, preventing premature restarts of slow-starting containers.

```yaml
startupProbe:
  httpGet:
    path: /startupz
    port: 8080
  failureThreshold: 30      # Allow up to 5 minutes to start (30 × 10s)
  periodSeconds: 10
  timeoutSeconds: 5
```

### Deep Health Checks

A deep health check endpoint (`/health` or `/health/detailed`) reports the status of all downstream dependencies. This is useful for dashboards, troubleshooting, and operations tools — but should **not** be used as a Kubernetes probe (it would cause cascading restarts on dependency failures).

**Example JSON response:**

```json
{
  "status": "degraded",
  "timestamp": "2024-01-15T10:30:00Z",
  "version": "1.4.2",
  "uptime_seconds": 86400,
  "checks": {
    "database": {
      "status": "healthy",
      "latency_ms": 3,
      "pool_size": 10,
      "pool_idle": 7
    },
    "redis": {
      "status": "healthy",
      "latency_ms": 1,
      "connected_clients": 3
    },
    "payment_service": {
      "status": "unhealthy",
      "error": "connection refused: payment-service:8080",
      "last_healthy": "2024-01-15T10:25:00Z"
    },
    "kafka": {
      "status": "healthy",
      "consumer_lag": 12,
      "partition_leaders_online": 3
    }
  }
}
```

### Health Check Implementation — Go

```go
package health

import (
    "context"
    "database/sql"
    "encoding/json"
    "net/http"
    "time"
)

type Status string

const (
    StatusHealthy   Status = "healthy"
    StatusDegraded  Status = "degraded"
    StatusUnhealthy Status = "unhealthy"
)

type CheckResult struct {
    Status    Status `json:"status"`
    LatencyMs int64  `json:"latency_ms,omitempty"`
    Error     string `json:"error,omitempty"`
}

type HealthResponse struct {
    Status    Status                 `json:"status"`
    Timestamp string                 `json:"timestamp"`
    Version   string                 `json:"version"`
    Checks    map[string]CheckResult `json:"checks"`
}

func DeepHealthHandler(db *sql.DB, version string) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        checks := make(map[string]CheckResult)
        overallStatus := StatusHealthy

        // Check database
        start := time.Now()
        ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
        defer cancel()
        if err := db.PingContext(ctx); err != nil {
            checks["database"] = CheckResult{Status: StatusUnhealthy, Error: err.Error()}
            overallStatus = StatusDegraded
        } else {
            checks["database"] = CheckResult{
                Status:    StatusHealthy,
                LatencyMs: time.Since(start).Milliseconds(),
            }
        }

        resp := HealthResponse{
            Status:    overallStatus,
            Timestamp: time.Now().UTC().Format(time.RFC3339),
            Version:   version,
            Checks:    checks,
        }

        statusCode := http.StatusOK
        if overallStatus == StatusUnhealthy {
            statusCode = http.StatusServiceUnavailable
        }

        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(statusCode)
        json.NewEncoder(w).Encode(resp)
    }
}
```

### Health Check Implementation — Python (FastAPI)

```python
from fastapi import FastAPI, Response
from sqlalchemy import text
from datetime import datetime, timezone
import time
import httpx

app = FastAPI()

@app.get("/health")
async def deep_health(response: Response):
    checks = {}
    overall_status = "healthy"

    # Check database
    start = time.monotonic()
    try:
        await db.execute(text("SELECT 1"))
        checks["database"] = {
            "status": "healthy",
            "latency_ms": round((time.monotonic() - start) * 1000),
        }
    except Exception as e:
        checks["database"] = {"status": "unhealthy", "error": str(e)}
        overall_status = "degraded"

    # Check downstream service
    start = time.monotonic()
    try:
        async with httpx.AsyncClient(timeout=2.0) as client:
            r = await client.get("http://payment-service:8080/livez")
            r.raise_for_status()
            checks["payment_service"] = {
                "status": "healthy",
                "latency_ms": round((time.monotonic() - start) * 1000),
            }
    except Exception as e:
        checks["payment_service"] = {"status": "unhealthy", "error": str(e)}
        overall_status = "degraded"

    if overall_status != "healthy":
        response.status_code = 503

    return {
        "status": overall_status,
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "version": settings.version,
        "checks": checks,
    }

@app.get("/livez")
async def liveness():
    """Fast liveness check — only verifies the process is alive."""
    return {"status": "alive"}

@app.get("/readyz")
async def readiness():
    """Readiness check — verifies the app can serve requests."""
    if not db_pool_initialized:
        return Response(status_code=503, content='{"status":"not ready"}')
    return {"status": "ready"}
```

---

## Dashboard Design Principles

### The First Question: Is the Service Healthy Right Now?

The top of every service dashboard must answer **"Is the service healthy right now?"** within 5 seconds of opening. Put the most important signal — current error rate, SLO status, and p99 latency — at the very top in large stat panels with colour thresholds (green/yellow/red).

### Dashboard Structure

```
┌─────────────────────────────────────────────────────────────┐
│  ROW 1: Service Health Summary (stat panels)                │
│  [SLO Status: 99.97%] [Error Rate: 0.03%] [p99: 245ms]    │
│  [Traffic: 1,234 RPS] [Uptime: 99.99%]                     │
├─────────────────────────────────────────────────────────────┤
│  ROW 2: RED Metrics (time-series graphs)                    │
│  [Request Rate over time] [Error Rate over time]            │
│  [p50/p95/p99 Latency over time]                           │
├─────────────────────────────────────────────────────────────┤
│  ROW 3: Resources (saturation)                              │
│  [CPU utilisation] [Memory utilisation] [Disk I/O]         │
│  [Network throughput] [Pod count / restarts]                │
├─────────────────────────────────────────────────────────────┤
│  ROW 4: Detailed / Debug (collapsed by default)             │
│  [Per-handler latency breakdown] [DB query times]           │
│  [Downstream service latency] [Cache hit/miss ratios]       │
└─────────────────────────────────────────────────────────────┘
```

### Panel Layout Recommendations

| Row | Panel Types | Guidance |
|-----|------------|----------|
| Summary | Stat, Gauge | Large numbers with colour thresholds; no time axis |
| RED Metrics | Time series | Show last 1h; include alert annotations |
| Resources | Time series, Gauge | Include resource limits as reference lines |
| Detailed | Table, Heatmap, Bar chart | Collapse by default; for deep investigation |

### Variable/Template Usage

Use Grafana variables to make dashboards reusable across services and environments:

```
Variables to define:
  $service     = label_values(http_requests_total, service)
  $namespace   = label_values(kube_pod_info, namespace)
  $interval    = ['1m', '5m', '15m', '30m', '1h']
  $quantile    = ['0.50', '0.95', '0.99']

Usage in PromQL:
  rate(http_requests_total{service="$service"}[$interval])
  histogram_quantile($quantile, ...)
```

### Avoid Common Dashboard Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| No time range defaults | Loaded with "last 6 hours" making panels slow | Default to "last 1 hour" |
| Averages instead of percentiles | Hides slow tail requests | Use `histogram_quantile` for latency |
| No alert annotations | Cannot correlate alerts to graph dips | Enable alert annotations on all time series panels |
| Too many panels (50+) | Cognitive overload, slow to load | Max 20 panels per dashboard; link to sub-dashboards |
| Hardcoded service names | Dashboard only works for one service | Use template variables |
| No units on axes | "Is that 200ms or 200s?" | Configure Grafana unit for every panel |
| Gauge without limits | 50 of what? | Always set min and max on gauges |
| Using rate() directly on gauges | Gauge has no `rate()` | Use `increase()` for counters |

---

## Observability as Code

Storing observability configuration — dashboards, alerts, recording rules — in version control provides the same benefits as infrastructure as code: auditability, reproducibility, and peer review.

### Grafonnet/Jsonnet for Grafana Dashboards

```jsonnet
// dashboards/order-service.jsonnet
local grafana = import 'grafonnet/grafana.libsonnet';
local dashboard = grafana.dashboard;
local graphPanel = grafana.graphPanel;
local prometheus = grafana.prometheus;

dashboard.new(
  'Order Service',
  tags=['microservices', 'order'],
  time_from='now-1h',
  refresh='30s',
)
.addPanel(
  graphPanel.new(
    'Request Rate',
    datasource='Prometheus',
    legend_show=true,
    fill=1,
  )
  .addTarget(
    prometheus.target(
      'sum(rate(http_requests_total{service="order-service"}[5m])) by (status_code)',
      legendFormat='{{status_code}}',
    )
  ),
  gridPos={x: 0, y: 0, w: 12, h: 8}
)
```

### Terraform for Alerting Rules

```hcl
# modules/prometheus-alerts/main.tf
resource "kubernetes_config_map" "order_service_alerts" {
  metadata {
    name      = "order-service-alerts"
    namespace = var.namespace
    labels = {
      "prometheus" = "kube-prometheus"
      "role"       = "alert-rules"
    }
  }

  data = {
    "order-service.yml" = yamlencode({
      groups = [{
        name = "order-service"
        rules = [
          {
            alert = "OrderServiceHighErrorRate"
            expr  = <<-PROMQL
              sum(rate(http_requests_total{
                service="order-service",
                status_code=~"5.."
              }[5m]))
              /
              sum(rate(http_requests_total{service="order-service"}[5m]))
              > 0.01
            PROMQL
            for  = "5m"
            labels = {
              severity = "critical"
              team     = "platform"
            }
            annotations = {
              summary     = "Order Service error rate > 1%"
              description = "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes."
              runbook_url = "https://wiki.internal/runbooks/order-service-errors"
            }
          }
        ]
      }]
    })
  }
}
```

### Helm Values for kube-prometheus-stack

```yaml
# values/observability.yaml — values for prometheus-community/kube-prometheus-stack
kube-prometheus-stack:
  prometheus:
    prometheusSpec:
      retention: 15d
      retentionSize: "50GB"
      resources:
        requests:
          cpu: 500m
          memory: 2Gi
        limits:
          memory: 8Gi
      storageSpec:
        volumeClaimTemplate:
          spec:
            storageClassName: fast-ssd
            accessModes: ["ReadWriteOnce"]
            resources:
              requests:
                storage: 100Gi

  grafana:
    adminPassword: "${GRAFANA_ADMIN_PASSWORD}"
    sidecar:
      dashboards:
        enabled: true        # Auto-load dashboards from ConfigMaps
        label: grafana_dashboard
    datasources:
      datasources.yaml:
        apiVersion: 1
        datasources:
          - name: Prometheus
            type: prometheus
            url: http://kube-prometheus-stack-prometheus:9090
            isDefault: true
          - name: Loki
            type: loki
            url: http://loki:3100

  alertmanager:
    config:
      global:
        resolve_timeout: 5m
        pagerduty_url: "https://events.pagerduty.com/v2/enqueue"
      route:
        group_by: ['alertname', 'cluster', 'service']
        group_wait: 30s
        group_interval: 5m
        repeat_interval: 12h
        receiver: 'default'
        routes:
          - match:
              severity: critical
            receiver: pagerduty-critical
      receivers:
        - name: pagerduty-critical
          pagerduty_configs:
            - routing_key: "${PAGERDUTY_KEY}"
```

### Benefits of Observability as Code

| Benefit | Description |
|---------|-------------|
| **Version control** | Every dashboard and alert change is audited with author and reason |
| **Code review** | Peers can review alert thresholds and dashboard logic before merging |
| **Reproducibility** | Spin up identical observability for staging, canary, and production |
| **Testing** | Alert rules can be unit-tested with `promtool test rules` |
| **Consistency** | Templates ensure all services follow the same dashboard structure |
| **Disaster recovery** | Rebuild entire observability stack from Git in minutes |

---

## Runbook Automation

### Automated Runbooks with PagerDuty Webhooks

Automated runbooks reduce MTTR (Mean Time To Recovery) by executing first-response actions automatically when an alert fires, before a human is paged.

```python
# runbooks/auto_responder.py
# Flask webhook handler for PagerDuty event webhooks
from flask import Flask, request, jsonify
import subprocess
import logging

app = Flask(__name__)
logger = logging.getLogger(__name__)

@app.route("/webhook/pagerduty", methods=["POST"])
def pagerduty_webhook():
    payload = request.json
    event_type = payload.get("event", {}).get("event_type")
    alert_name = payload.get("event", {}).get("data", {}).get("title", "")

    if event_type == "incident.triggered":
        logger.info("alert triggered", extra={"alert": alert_name})
        dispatch_runbook(alert_name, payload)

    return jsonify({"status": "ok"})

def dispatch_runbook(alert_name: str, payload: dict):
    runbooks = {
        "PodOOMKilled": runbook_restart_oomkilled_pod,
        "HighDiskUsage": runbook_cleanup_old_logs,
        "DatabaseConnectionPoolExhausted": runbook_bounce_connection_pool,
    }
    if fn := runbooks.get(alert_name):
        fn(payload)
    else:
        logger.info("no automated runbook for alert", extra={"alert": alert_name})
```

### Self-Healing Scripts

```bash
#!/bin/bash
# runbooks/restart-oomkilled-pods.sh
# Restart pods in CrashLoopBackOff due to OOMKilled

NAMESPACE=${1:-default}
MAX_RESTARTS=${2:-5}

echo "Checking for OOMKilled pods in namespace: $NAMESPACE"

kubectl get pods -n "$NAMESPACE" \
    -o jsonpath='{range .items[*]}{.metadata.name} {.status.containerStatuses[0].lastState.terminated.reason}{"\n"}{end}' \
  | grep "OOMKilled" \
  | while read -r pod reason; do
      restarts=$(kubectl get pod "$pod" -n "$NAMESPACE" \
          -o jsonpath='{.status.containerStatuses[0].restartCount}')

      if [[ "$restarts" -lt "$MAX_RESTARTS" ]]; then
          echo "Deleting OOMKilled pod $pod (restarts: $restarts)"
          kubectl delete pod "$pod" -n "$NAMESPACE"
          echo "Pod $pod deleted; Kubernetes will reschedule it"
      else
          echo "Pod $pod has restarted $restarts times — escalating to human"
          # Page on-call engineer instead
          curl -X POST https://events.pagerduty.com/v2/enqueue \
            -H "Content-Type: application/json" \
            -d "{\"routing_key\": \"$PD_KEY\", \"event_action\": \"trigger\", \"payload\": {\"summary\": \"Pod $pod exceeds OOMKill threshold\", \"severity\": \"critical\"}}"
      fi
  done
```

### Example: Auto-Restart Pod on OOMKilled

```go
// go/runbooks/oom_responder.go
package runbooks

import (
    "context"
    "fmt"
    "log"

    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
)

const maxAutoRestarts = 3

func HandleOOMKilledPods(ctx context.Context, client kubernetes.Interface, namespace string) error {
    pods, err := client.CoreV1().Pods(namespace).List(ctx, metav1.ListOptions{})
    if err != nil {
        return fmt.Errorf("listing pods: %w", err)
    }

    for _, pod := range pods.Items {
        for _, cs := range pod.Status.ContainerStatuses {
            if cs.LastTerminationState.Terminated != nil &&
                cs.LastTerminationState.Terminated.Reason == "OOMKilled" {

                if cs.RestartCount < maxAutoRestarts {
                    log.Printf("auto-restarting OOMKilled pod %s (restarts: %d)",
                        pod.Name, cs.RestartCount)
                    if err := client.CoreV1().Pods(namespace).Delete(
                        ctx, pod.Name, metav1.DeleteOptions{}); err != nil {
                        log.Printf("error deleting pod %s: %v", pod.Name, err)
                    }
                } else {
                    log.Printf("pod %s exceeded OOM restart threshold — escalating", pod.Name)
                    // Trigger PagerDuty escalation
                    notifyOnCall(pod.Name, cs.RestartCount)
                }
            }
        }
    }
    return nil
}
```

---

## Next Steps

1. **Apply the Golden Signals** — Instrument at least one service in your stack with all four signals and build a dashboard.
2. **Audit Cardinality** — Run `promtool tsdb analyze` or check your Prometheus TSDB status page; reduce your top-5 highest cardinality metrics.
3. **Add Correlation IDs** — Implement correlation ID middleware across your service boundary entry points.
4. **Review Anti-Patterns** — Read [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md) and audit your systems against the checklist.
5. **Define SLOs** — If you have not defined SLOs, start with [07-SLIS-SLOS-SLAS.md](07-SLIS-SLOS-SLAS.md).
6. **Codify Observability** — Move dashboards and alerts to version control using Grafonnet or Terraform.
7. **Follow the Learning Path** — Use [LEARNING-PATH.md](LEARNING-PATH.md) for a structured curriculum.

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-01-01 | Initial document — Four Golden Signals, RED, USE, instrumentation standards | Platform Team |
| 1.1 | 2025-01-15 | Added cardinality management, correlation IDs, sampling strategies | Observability Guild |
| 1.2 | 2025-02-01 | Added health check patterns, dashboard design principles | SRE Team |
| 1.3 | 2025-02-15 | Added observability as code, runbook automation, cost optimization | Platform Team |
