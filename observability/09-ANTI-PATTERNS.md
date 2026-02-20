# Observability Anti-Patterns

A catalogue of the most common observability mistakes — what they look like, why they are harmful, and exactly how to fix them. Use this document as a code-review checklist, a pre-production gate, or a team learning resource.

---

## Table of Contents

- [Introduction](#introduction)
- [Anti-Patterns Summary Table](#anti-patterns-summary-table)
- [1. Alert Fatigue](#1-alert-fatigue)
- [2. Monitoring Everything](#2-monitoring-everything)
- [3. Missing Distributed Tracing](#3-missing-distributed-tracing)
- [4. High-Cardinality Labels](#4-high-cardinality-labels)
- [5. Logging Too Much](#5-logging-too-much)
- [6. Logging Too Little](#6-logging-too-little)
- [7. Missing Structured Logging](#7-missing-structured-logging)
- [8. Monitoring Causes Instead of Symptoms](#8-monitoring-causes-instead-of-symptoms)
- [9. Not Having SLOs](#9-not-having-slos)
- [10. Vendor Lock-in](#10-vendor-lock-in)
- [11. Treating Observability as an Afterthought](#11-treating-observability-as-an-afterthought)
- [12. Missing Correlation IDs](#12-missing-correlation-ids)
- [13. Using Averages Instead of Percentiles](#13-using-averages-instead-of-percentiles)
- [14. Not Testing Alerts](#14-not-testing-alerts)
- [Observability Maturity Anti-Patterns](#observability-maturity-anti-patterns)
- [Quick Reference Checklist](#quick-reference-checklist)
- [Next Steps](#next-steps)
- [Version History](#version-history)

---

## Introduction

### Why Anti-Patterns Matter

Anti-patterns are recurring practices that seem reasonable at first glance but create significant problems over time. In observability, the consequences of anti-patterns are typically felt acutely during an incident — the exact moment when your observability tools need to work flawlessly.

The patterns documented here represent real failures observed across production systems. Each one is:

- **Seductive** — it felt like the right approach when implemented
- **Harmful** — it creates noise, blindness, cost, or incident prolongation
- **Fixable** — there is a well-understood better approach

### How to Use This Document

1. **Pre-production review:** Use the [Quick Reference Checklist](#quick-reference-checklist) before releasing a new service
2. **Code review:** Reference specific sections when reviewing observability-related PRs
3. **Incident retros:** After an incident, check which anti-patterns contributed to MTTR
4. **Team onboarding:** Assign this document to new engineers before they touch production observability configuration
5. **Periodic audit:** Run through the checklist quarterly for existing services

---

## Anti-Patterns Summary Table

| # | Anti-Pattern | Category | Severity |
|---|--------------|----------|----------|
| 1 | Alert Fatigue | Alerting | 🔴 Critical |
| 2 | Monitoring Everything | Metrics | 🟠 High |
| 3 | Missing Distributed Tracing | Tracing | 🟠 High |
| 4 | High-Cardinality Labels | Metrics | 🔴 Critical |
| 5 | Logging Too Much | Logging | 🟠 High |
| 6 | Logging Too Little | Logging | 🟠 High |
| 7 | Missing Structured Logging | Logging | 🟠 High |
| 8 | Monitoring Causes Instead of Symptoms | Alerting | 🔴 Critical |
| 9 | Not Having SLOs | Process | 🔴 Critical |
| 10 | Vendor Lock-in | Architecture | 🟡 Medium |
| 11 | Treating Observability as an Afterthought | Process | 🔴 Critical |
| 12 | Missing Correlation IDs | Tracing / Logging | 🟠 High |
| 13 | Using Averages Instead of Percentiles | Metrics | 🟠 High |
| 14 | Not Testing Alerts | Alerting | 🟠 High |

---

## 1. Alert Fatigue

### Problem

Alert fatigue occurs when engineers receive so many alerts that they begin ignoring or silencing them habitually. When the real critical alert fires during an incident, it is drowned in noise — or worse, the on-call engineer has trained themselves to dismiss alerts without reading them.

### Example of the Bad Pattern

```yaml
# alerting/all-the-things.yml — 200+ alerts, all sent to the same Slack channel
groups:
  - name: everything
    rules:
      - alert: HighCPU
        expr: cpu_usage > 50
        # No severity. No routing. No runbook. No for: duration.
      - alert: MemoryAbove60Percent
        expr: memory_usage > 0.6
      - alert: DiskAbove40Percent
        expr: disk_usage > 0.4
      - alert: AnyHTTP4xx
        expr: http_requests_total{status=~"4.."} > 0
      - alert: AnyHTTP5xx
        expr: http_requests_total{status=~"5.."} > 0
      - alert: PodRestarted
        expr: kube_pod_container_status_restarts_total > 0
      - alert: HighLatency
        expr: http_request_duration_seconds > 0.1
      # ... 193 more alerts like this
```

**Symptoms of alert fatigue:**
- On-call team silences whole alert groups permanently
- P1 incidents go unnoticed for 10+ minutes
- Slack channel has 500+ unread alert messages every Monday morning
- Engineers turn off mobile notifications for the alerting channel
- Post-mortems note "we saw the alert but assumed it was noise"

### Why It's Harmful

- Engineers stop responding to pages, increasing MTTR dramatically
- Real critical alerts are buried in the noise
- Trust in the alerting system erodes — eventually nobody acts on any alert
- On-call burnout leads to engineer attrition
- 80% of informational alerts add zero operational value

### Recommended Fix

```yaml
# alerting/order-service-alerts.yml — symptom-based, tiered, routed
groups:
  - name: order-service.critical
    rules:
      # Alert on user impact, not infrastructure metrics
      - alert: OrderServiceErrorRateHigh
        expr: |
          sum(rate(http_requests_total{
            service="order-service",
            status_code=~"5.."
          }[5m]))
          /
          sum(rate(http_requests_total{service="order-service"}[5m]))
          > 0.01
        for: 5m
        labels:
          severity: critical
          team: platform
          service: order-service
        annotations:
          summary: "Order Service error rate exceeds 1%"
          description: "Error rate is {{ $value | humanizePercentage }}. Potential user impact."
          runbook_url: "https://wiki.internal/runbooks/order-service-errors"

  - name: order-service.warning
    rules:
      - alert: OrderServiceLatencyHigh
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket{
              service="order-service"
            }[5m])) by (le)
          ) > 1.0
        for: 10m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Order Service p99 latency > 1s"
          runbook_url: "https://wiki.internal/runbooks/order-service-latency"
```

**Alertmanager routing:**

```yaml
# alertmanager.yml
route:
  group_by: ['service', 'alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: slack-platform-warnings

  routes:
    - match:
        severity: critical
      receiver: pagerduty-critical
      continue: true       # Also send to Slack

    - match:
        severity: warning
      receiver: slack-platform-warnings
      repeat_interval: 24h # Warnings at most once per day

receivers:
  - name: pagerduty-critical
    pagerduty_configs:
      - routing_key: "${PAGERDUTY_KEY}"
  - name: slack-platform-warnings
    slack_configs:
      - channel: "#platform-alerts"
```

### Quick Check

- [ ] All alerts have `severity: critical | warning | info` labels
- [ ] Critical alerts page on-call via PagerDuty/OpsGenie; warnings go to chat only
- [ ] Every alert has a `runbook_url` annotation
- [ ] All alerts have a minimum `for: 5m` (no flap-alerting)
- [ ] Weekly alert review: dismiss or fix any alert that fires and requires no action

---

## 2. Monitoring Everything

### Problem

Teams configure Prometheus to scrape every available metric endpoint and store everything without filtering. This results in hundreds of thousands of time series, most of which are never queried, driving up memory, storage, and query costs while making useful signals hard to find.

### Example of the Bad Pattern

```yaml
# prometheus.yml — scraping everything without discrimination
scrape_configs:
  - job_name: "all-services"
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - action: keep
        source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        regex: "true"
    # No metric_relabel_configs — storing ALL 50,000+ series
```

This configuration results in:

```
# 50,000+ active time series including:
go_memstats_alloc_bytes_total
go_memstats_buck_hash_sys_bytes
go_memstats_frees_total
go_memstats_gc_cpu_fraction
go_memstats_gc_sys_bytes
go_memstats_heap_alloc_bytes
go_memstats_heap_idle_bytes
... (40+ go runtime metrics per service × 20 services = 800 series nobody looks at)
process_cpu_seconds_total
process_max_fds
process_open_fds
process_resident_memory_bytes
process_start_time_seconds
process_virtual_memory_bytes
process_virtual_memory_max_bytes
... (7 process metrics × 20 services = 140 more)
```

### Why It's Harmful

- Prometheus OOM-kills on insufficient memory allocation
- Query times increase as TSDB grows (every PromQL scan takes longer)
- Storage costs scale linearly with series count
- Engineers cannot find signal in the noise when debugging
- Every dashboard load is slow due to scanning thousands of irrelevant series
- Ingestion backpressure causes scrape failures

### Recommended Fix

```yaml
# prometheus.yml — focused metrics with relabeling
scrape_configs:
  - job_name: "order-service"
    static_configs:
      - targets: ["order-service:8080"]
    metric_relabel_configs:
      # Drop all Go runtime internals (rarely actionable in production)
      - source_labels: [__name__]
        regex: "go_(memstats_buck_hash|memstats_gc_cpu|memstats_heap_released|memstats_next_gc|memstats_last_gc|memstats_other_sys|memstats_stack_sys|memstats_mspan_sys|memstats_mcache_sys).*"
        action: drop

      # Drop internal Prometheus process metrics from app
      - source_labels: [__name__]
        regex: "process_(max_fds|virtual_memory_max_bytes).*"
        action: drop

      # Drop specific high-cardinality label from a noisy metric
      - source_labels: [__name__]
        regex: "http_request_duration_seconds.*"
        target_label: request_id
        replacement: ""
        action: replace
```

**Identify the vital few metrics (RED + Golden Signals):**

```
Essential metrics per service (aim for < 20):
  http_requests_total          ← Traffic + Errors
  http_request_duration_seconds_bucket ← Latency (histogram)
  process_resident_memory_bytes ← Saturation
  process_cpu_seconds_total    ← Saturation
  <app>_errors_total           ← Business errors
  <app>_queue_depth            ← Saturation
  <app>_cache_hits_total       ← Efficiency
  <app>_cache_misses_total     ← Efficiency
```

### Quick Check

- [ ] Total active time series < 500,000 per Prometheus instance
- [ ] `metric_relabel_configs` used to drop unused metrics on all scrape jobs
- [ ] Go runtime and process metrics reviewed — only keep what you actively dashboard
- [ ] New metrics require team review before merging (recorded in ADR or PR)
- [ ] `promtool tsdb analyze` run monthly to identify cardinality growth

---

## 3. Missing Distributed Tracing

### Problem

In a distributed system, a single user request touches multiple services. Without distributed tracing, diagnosing latency or errors requires manually correlating log timestamps across services — a slow, error-prone process that significantly increases MTTR.

### Example of the Bad Pattern

```
# Debugging a "slow checkout" complaint without tracing:
# Service A log (order-service):
2024-01-15 10:30:00.123 INFO  Request received: POST /checkout user=cust-123
2024-01-15 10:30:02.891 INFO  Checkout complete: order-456

# Service B log (payment-service):
2024-01-15 10:30:00.890 INFO  Payment request received
2024-01-15 10:30:01.456 INFO  Payment authorized
2024-01-15 10:30:01.460 INFO  Payment confirmed

# Service C log (inventory-service):
2024-01-15 10:30:00.145 INFO  Reservation request received
2024-01-15 10:30:02.871 INFO  Reservation confirmed — waited 2.7s for DB lock

# Investigation:
# Engineer must manually correlate timestamps to figure out:
# - Inventory service caused the 2.7s delay (DB lock wait)
# - No way to confirm these log lines are from the same request
# - Could be coincidental timing from different requests
# MTTR impact: 45 minutes to find a 2-line bug
```

### Why It's Harmful

- Cannot determine which service caused a latency spike
- Cannot trace the path of a failed request across service boundaries
- Manually correlating timestamps is unreliable and slow
- Impossible to see waterfall (what ran in parallel vs serial)
- Cannot identify which downstream dependency is the bottleneck
- P1 incidents take 10× longer to resolve

### Recommended Fix

```python
# Auto-instrumentation with OpenTelemetry (Python)
# requirements.txt additions:
# opentelemetry-api
# opentelemetry-sdk
# opentelemetry-instrumentation-fastapi
# opentelemetry-instrumentation-requests
# opentelemetry-exporter-otlp

# main.py
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
import structlog

# Configure tracing
provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(OTLPSpanExporter(endpoint="http://otel-collector:4317"))
)
trace.set_tracer_provider(provider)

# Auto-instrument FastAPI and outgoing HTTP
FastAPIInstrumentor.instrument_app(app)
RequestsInstrumentor().instrument()

# Inject trace ID into all log lines
@app.middleware("http")
async def add_trace_id_to_logs(request: Request, call_next):
    span = trace.get_current_span()
    ctx = span.get_span_context()
    structlog.contextvars.bind_contextvars(
        trace_id=format(ctx.trace_id, "032x"),
        span_id=format(ctx.span_id, "016x"),
    )
    response = await call_next(request)
    structlog.contextvars.clear_contextvars()
    return response
```

**Result:** Now the same checkout request produces:

```json
{"timestamp": "2024-01-15T10:30:00.123Z", "service": "order-service",
 "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736", "msg": "Request received"}

{"timestamp": "2024-01-15T10:30:02.871Z", "service": "inventory-service",
 "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736", "msg": "DB lock wait resolved",
 "wait_ms": 2726}
```

One `trace_id` query in Jaeger or Grafana Tempo shows the entire waterfall in seconds.

### Quick Check

- [ ] All services emit traces with trace context propagation
- [ ] Trace IDs are included in every log line
- [ ] A trace backend (Jaeger, Tempo, Zipkin) is deployed and retained for ≥ 7 days
- [ ] Service dependency map is visible in the trace backend
- [ ] p99 latency breakdown by service is dashboarded using traces

---

## 4. High-Cardinality Labels

### Problem

Adding high-cardinality values — such as user IDs, request IDs, or raw URL paths — as Prometheus metric labels creates one time series per unique value, rapidly exhausting Prometheus memory and causing OOM crashes or severe query degradation.

### Example of the Bad Pattern

```go
// Bad: labeling metrics with high-cardinality values
var requestCounter = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total HTTP requests",
    },
    // These labels create CATASTROPHIC cardinality:
    []string{"user_id", "request_id", "session_id", "url_path"},
    // user_id: 1,000,000 users
    // request_id: unique per request = billions
    // session_id: millions of sessions
    // url_path: /api/users/123, /api/users/456 ... unbounded
)

// In handler — one new time series combination per request!
requestCounter.With(prometheus.Labels{
    "user_id":    userID,        // "user-12345678"
    "request_id": requestID,    // "req-a1b2c3d4e5f6"
    "session_id": sessionID,    // "sess-987654321"
    "url_path":   r.URL.Path,   // "/api/users/12345678/orders/987"
}).Inc()
```

**The math:**

```
1,000,000 users × billions of request_ids = system crash within minutes
Even with just user_id: 1,000,000 series just for this counter × many metrics
= Prometheus OOM
```

### Why It's Harmful

- Prometheus runs out of memory and crashes (or is OOM-killed by Kubernetes)
- Each new time series requires ~3KB of RAM in Prometheus
- Query performance degrades: scanning 10M series per query
- TSDB compaction takes hours instead of minutes
- Remote write to Cortex/Thanos causes backpressure and data loss
- Recovery requires a full Prometheus restart and TSDB cleanup

### Recommended Fix

```go
// Good: only low-cardinality labels on metrics
var requestCounter = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total HTTP requests",
    },
    // Max cardinality: ~5 × ~10 × ~50 = 2,500 series — safe
    []string{"method", "status_code", "handler"},
)

// Normalise URL to template — extract from routing layer
// /api/users/12345678 → /api/users/{user_id}
func handlerTemplate(r *http.Request) string {
    // Use chi/gin/echo route pattern, not raw path
    return chi.RouteContext(r.Context()).RoutePattern()
}

// For user-level analysis, use trace exemplars — NOT labels
span := trace.SpanFromContext(r.Context())
requestCounter.With(prometheus.Labels{
    "method":      r.Method,
    "status_code": strconv.Itoa(statusCode),
    "handler":     handlerTemplate(r),
}).Inc()
```

**Use exemplars to link metrics to specific requests:**

```go
// Attach high-cardinality trace context to the histogram observation
// without creating a time series per trace
histogram.With(labels).(prometheus.ExemplarObserver).ObserveWithExemplar(
    duration.Seconds(),
    prometheus.Labels{
        "trace_id": traceID,  // High cardinality — but stored as exemplar, not label
        "user_id":  userID,
    },
)
```

### Quick Check

- [ ] No label has more than ~100 unique values (check with `count by (label_name)(metric_name)`)
- [ ] `user_id`, `request_id`, `session_id` are NEVER Prometheus labels
- [ ] Raw `url_path` is NEVER a Prometheus label — use route template
- [ ] High-cardinality data uses traces or log fields instead of metric labels
- [ ] `promtool tsdb analyze` shows no single label with > 10,000 values

---

## 5. Logging Too Much

### Problem

Debug-level logging in production, or logging every request/response body, generates enormous log volumes that cost money to store and make relevant log lines nearly impossible to find. I/O-bound logging in tight loops also degrades application performance.

### Example of the Bad Pattern

```python
# Bad: DEBUG logging in production, logging full request/response bodies
import logging

logging.basicConfig(level=logging.DEBUG)  # DEBUG in production
logger = logging.getLogger(__name__)

@app.route("/api/orders")
def list_orders():
    logger.debug(f"Starting list_orders handler")
    logger.debug(f"Request headers: {dict(request.headers)}")   # may contain tokens
    logger.debug(f"Request body: {request.get_data()}")         # may be MB of data
    logger.debug(f"Database connection pool status: {pool.status()}")

    orders = db.query("SELECT * FROM orders")

    for order in orders:
        logger.debug(f"Processing order: {order}")  # logs every row

    response_data = [o.to_dict() for o in orders]
    logger.debug(f"Response: {response_data}")  # may be MB

    return jsonify(response_data)
```

**Production impact:**
- 1,000 RPS × 10 DEBUG lines per request = 10,000 log lines/second
- At 200 bytes average: 2MB/second = 172GB/day of debug logs
- Loki or Elasticsearch ingestion bill: ~$500–$2,000/month for a single service

### Why It's Harmful

- Log storage and ingestion costs 10–100× more than necessary
- Signal-to-noise ratio drops to near zero
- Log pipeline (Fluentd/Filebeat) becomes the bottleneck
- Request body logging creates **PII exposure** and compliance violations
- Credential leakage (tokens in headers logged as DEBUG)
- I/O blocking on log writes degrades application latency

### Recommended Fix

```python
# Good: environment-based log levels, structured, no sensitive data
import os
import structlog
import random

# Set log level from environment — DEBUG only in local dev
LOG_LEVEL = os.environ.get("LOG_LEVEL", "INFO").upper()

structlog.configure(
    wrapper_class=structlog.make_filtering_bound_logger(
        getattr(logging, LOG_LEVEL, logging.INFO)
    ),
)
logger = structlog.get_logger()

@app.route("/api/orders")
def list_orders():
    # INFO: one line per request at service boundary — always useful
    logger.info("list_orders",
                user_id=g.user_id,
                correlation_id=g.correlation_id)

    orders = db.query("SELECT * FROM orders WHERE user_id = ?", g.user_id)

    # Sample debug logging — only 1% of the time in production
    if LOG_LEVEL == "DEBUG" or random.random() < 0.01:
        logger.debug("orders_fetched", count=len(orders))

    logger.info("list_orders_complete",
                order_count=len(orders),
                correlation_id=g.correlation_id)
    return jsonify([o.to_dict() for o in orders])
```

**Fluent Bit filter to drop health check and metrics probe logs:**

```ini
[FILTER]
    Name    grep
    Match   kubernetes.*
    Exclude log /(healthz|readyz|metrics)
```

### Quick Check

- [ ] `LOG_LEVEL=INFO` (or WARN) in all production deployments
- [ ] DEBUG only enabled in local development or via feature flag
- [ ] Request/response bodies are NEVER logged (PII + cost concern)
- [ ] HTTP headers are NEVER logged in full (may contain Bearer tokens)
- [ ] Log volume is budgeted and monitored in the observability cost dashboard
- [ ] Health check endpoints are filtered out of log shippers

---

## 6. Logging Too Little

### Problem

Some teams over-correct and suppress nearly all logging, or only log errors without context. Silent failures — where operations silently return incorrect results, or external calls fail without logging — make production debugging nearly impossible.

### Example of the Bad Pattern

```go
// Bad: only logging errors, no context, no service boundary logging
func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    // No logging at all at entry point

    order, err := s.db.InsertOrder(ctx, req)
    if err != nil {
        // Just the error string — no context about what was being attempted
        log.Printf("error: %v", err)
        return nil, err
    }

    // Silent call to payment service — if this returns wrong data, we never know
    paymentResult, _ := s.paymentClient.Authorize(ctx, order.ID, req.Amount)
    // The _ discards the error — silent failure

    // No logging on success path — cannot see operational flow

    return order, nil
}
```

**Consequences:**
- Payment authorization failures are silently dropped
- Cannot tell from logs whether a request was received
- No way to correlate which requests led to incorrect state
- Debugging requires adding logging, redeploying, and reproducing the issue

### Why It's Harmful

- Silent failures become invisible until a user complains
- Cannot determine if a specific event occurred without querying the database
- MTTR increases dramatically — the first step is always "add logging and reproduce"
- Business logic bugs are undetectable for days or weeks
- Compliance audit trails are missing

### Recommended Fix

```go
// Good: structured logging at INFO for important operations, always log service boundaries
func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {
    log := s.logger.With(
        zap.String("correlation_id", middleware.GetCorrelationID(ctx)),
        zap.String("customer_id", req.CustomerID),
        zap.String("handler", "CreateOrder"),
    )

    // Log at service entry point
    log.Info("creating order",
        zap.Float64("amount", req.Amount),
        zap.Int("item_count", len(req.Items)),
    )

    order, err := s.db.InsertOrder(ctx, req)
    if err != nil {
        // Log error with full context
        log.Error("failed to insert order",
            zap.Error(err),
            zap.String("customer_id", req.CustomerID),
        )
        return nil, fmt.Errorf("inserting order: %w", err)
    }

    log.Info("order inserted", zap.String("order_id", order.ID))

    // Always handle and log errors from external calls
    paymentResult, err := s.paymentClient.Authorize(ctx, order.ID, req.Amount)
    if err != nil {
        log.Error("payment authorization failed",
            zap.Error(err),
            zap.String("order_id", order.ID),
        )
        // Compensate or return error — never silently discard
        return nil, fmt.Errorf("authorizing payment: %w", err)
    }

    log.Info("order created successfully",
        zap.String("order_id", order.ID),
        zap.String("payment_id", paymentResult.PaymentID),
    )
    return order, nil
}
```

### Quick Check

- [ ] Every service boundary entry point logs at INFO with request context
- [ ] Every external call (DB, API, queue) has error handling AND error logging
- [ ] No `_` error discards on non-trivial operations (enforce via linter)
- [ ] Business-significant events (order created, payment processed) are logged at INFO
- [ ] Every error log includes enough context to reproduce the problem

---

## 7. Missing Structured Logging

### Problem

Unstructured, printf-style log messages are human-readable but machine-unreadable. Log aggregation systems (Loki, Elasticsearch, Splunk) cannot parse arbitrary strings efficiently, forcing full-text search instead of field-level indexing — which is slow, expensive, and error-prone.

### Example of the Bad Pattern

```python
# Bad: printf-style unstructured logs
import logging
logger = logging.getLogger(__name__)

def process_order(user_id, order_id, amount):
    logger.info(f"User {user_id} placed order {order_id} for ${amount} at {datetime.now()}")
    logger.error(f"Failed to process payment for order {order_id}: connection timeout after 30s")
    logger.warning(f"Retry {retry_count} of 3 for order {order_id}")
```

**Output:**

```
2024-01-15 10:30:00,123 INFO User 12345 placed order ord-789 for $99.99 at 2024-01-15 10:30:00.123
2024-01-15 10:30:31,456 ERROR Failed to process payment for order ord-789: connection timeout after 30s
```

**Searching for all orders by user 12345:**

```
# Full-text search (slow, expensive, fragile):
grep "User 12345" /var/log/app.log

# What if the format changes slightly?
# "User 12345" vs "user_id=12345" vs "userId: 12345"
# Three different grep patterns needed — and they all break on log format changes
```

### Why It's Harmful

- Cannot reliably filter logs by field (user_id, order_id, error type)
- Full-text search in Elasticsearch is 10–100× more expensive than index lookup
- Log formats vary per developer; automated parsing requires complex regex
- Cannot build dashboards or alerts from unstructured log fields
- Sensitive data (PII) buried in log strings is harder to redact
- Log parsing fragility — any format change breaks log ingestion pipelines

### Recommended Fix

```python
# Good: structured JSON logging with consistent fields
import structlog
import logging

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.stdlib.add_log_level,
        structlog.stdlib.add_logger_name,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.JSONRenderer(),     # JSON output
    ],
    wrapper_class=structlog.BoundLogger,
    context_class=dict,
    logger_factory=structlog.PrintLoggerFactory(),
)

logger = structlog.get_logger()

def process_order(user_id, order_id, amount):
    logger.info("order_placed",
                user_id=user_id,
                order_id=order_id,
                amount_usd=amount)

    logger.error("payment_failed",
                 order_id=order_id,
                 error="connection timeout",
                 timeout_seconds=30)

    logger.warning("payment_retry",
                   order_id=order_id,
                   retry_number=retry_count,
                   max_retries=3)
```

**Output (machine-readable, field-indexed):**

```json
{"timestamp": "2024-01-15T10:30:00.123Z", "level": "info", "event": "order_placed",
 "user_id": "12345", "order_id": "ord-789", "amount_usd": 99.99}

{"timestamp": "2024-01-15T10:30:31.456Z", "level": "error", "event": "payment_failed",
 "order_id": "ord-789", "error": "connection timeout", "timeout_seconds": 30}
```

**Searching in Loki/Elasticsearch:**

```logql
# Loki LogQL — exact field lookup
{service="order-service"} | json | user_id = "12345"

# Elasticsearch
GET /logs/_search
{ "query": { "term": { "user_id.keyword": "12345" } } }
```

### Quick Check

- [ ] All log output is valid JSON (or structured key=value for logfmt)
- [ ] Log fields are consistent: `level`, `timestamp`, `service`, `event`, `correlation_id`
- [ ] No `f"..."` string interpolation for log messages (use structured fields)
- [ ] Log parsing configuration in the log shipper is not using regex
- [ ] Structured log fields are documented so new developers know what to include

---

## 8. Monitoring Causes Instead of Symptoms

### Problem

Infrastructure metrics (CPU, memory, disk) describe *causes* of problems, not the problems themselves. Alerting directly on infrastructure metrics creates noise (CPU spikes may be harmless) and misses real user impact (error rate can spike with normal CPU).

### Example of the Bad Pattern

```yaml
# Bad: cause-based alerts — monitoring infrastructure, not user experience
groups:
  - name: infrastructure-causes
    rules:
      - alert: HighCPU
        expr: avg(cpu_utilisation) > 0.80   # CPU cause
        for: 5m
        # Problem: CPU spike during batch job is harmless
        # Problem: Service can have 20% CPU but 50% error rate

      - alert: HighMemory
        expr: memory_utilisation > 0.75      # Memory cause
        for: 5m
        # Problem: Memory at 80% is fine for most apps

      - alert: DiskAlmostFull
        expr: disk_free_ratio < 0.15         # Disk cause
        for: 1m
        # OK: this one is actually valid as a standalone alert

      - alert: ManyGoroutines
        expr: go_goroutines > 1000           # Implementation detail
        for: 5m
        # Problem: 1000 goroutines may be perfectly normal
```

**The gap in this approach:**

```
CPU = 10%      ← looks healthy
Error rate = 15% ← users are failing — no alert fires!

CPU = 85%      ← alert fires, wakes on-call at 3am
Error rate = 0.01% ← everything is fine, users aren't affected
```

### Why It's Harmful

- False positives: alerts fire when users are fine (CPU spike during batch processing)
- False negatives: users are failing but no alert fires (low CPU + high error rate)
- On-call engineers fix the wrong thing (tuning CPU when error rate is the issue)
- Creates alert fatigue (see Anti-Pattern #1) from harmless infrastructure fluctuations
- Violates the "alert on what users experience" principle from Google SRE

### Recommended Fix

```yaml
# Good: symptom-based primary alerts, cause-based as secondary context
groups:
  - name: symptoms-primary
    rules:
      # SYMPTOM: users are experiencing errors (5xx)
      - alert: UserFacingErrors
        expr: |
          sum(rate(http_requests_total{status_code=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "1%+ of user requests are failing"
          runbook_url: "https://wiki/runbooks/high-error-rate"

      # SYMPTOM: users are experiencing slow responses
      - alert: UserFacingHighLatency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 2.0
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "p99 latency > 2s — users experiencing slowness"

  - name: causes-secondary
    rules:
      # CAUSE: CPU saturation — informational, not alerting
      # Use this on dashboards, not as primary alerts
      - alert: CPUSaturationWarning
        expr: avg(cpu_utilisation) > 0.85
        for: 15m
        labels:
          severity: warning     # Warning, not critical
        annotations:
          summary: "CPU saturation — may cause latency in the near future"
          # Does not page on-call by itself
```

**Decision tree:**

```
Is a user experiencing a worse outcome?
    YES → Symptom alert (critical, pages on-call)
    NO  → Cause metric (warning, Slack only, or dashboard only)
```

### Quick Check

- [ ] Primary on-call alerts are based on error rate, latency, or availability
- [ ] CPU/memory alerts are severity: warning (Slack, not page)
- [ ] Each alert answers "how is the user affected?"
- [ ] Cause metrics are on dashboards for context, not as primary alert conditions
- [ ] Alert runbook starts with "impact to users" before "how to investigate"

---

## 9. Not Having SLOs

### Problem

Without defined Service Level Objectives, every threshold decision is arbitrary, alerts have no business context, and "is the system healthy?" has no objective answer. Teams argue about whether a 5% error rate is acceptable instead of comparing to an agreed target.

### Example of the Bad Pattern

```yaml
# Bad: arbitrary threshold alerts with no SLO basis
- alert: ErrorRateHigh
  expr: error_rate > 0.05  # Why 5%? Who decided this? What's the user impact?
  # No SLO defined. No error budget. No burn rate.
  # No relationship between this threshold and business requirements.
  # "Monitored" but no agreed definition of acceptable service quality.

# Symptoms of missing SLOs in a system:
# - "The system is monitored" (but nobody knows what 'healthy' means)
# - Every incident is debated: "Is this actually a problem?"
# - Different team members have different expectations for reliability
# - Alert thresholds were set once in 2019 and never revisited
# - No objective way to prioritise reliability work vs feature work
```

### Why It's Harmful

- Cannot objectively answer "is the service healthy?" post-incident
- Alert thresholds are arbitrary and disconnected from user impact
- No way to quantify reliability debt or justify reliability investment
- Every incident response is reactive; no proactive reliability budgeting
- Cannot make data-driven decisions about deployment risk
- User experience expectations are unmanaged

### Recommended Fix

```yaml
# Good: SLO-based alerting with error budget burn rate
# Step 1: Define the SLO (in a document, not just in Prometheus)
# Service: Order API
# SLO: 99.9% of checkout requests complete successfully within 2 seconds
#      over a 30-day rolling window
# Error budget: 0.1% = 43.2 minutes of downtime per 30 days

# Step 2: Recording rules for SLO tracking
groups:
  - name: slo-recording
    rules:
      - record: job:checkout_request_success:ratio_rate5m
        expr: |
          1 - (
            sum(rate(http_requests_total{
              service="order-service",
              handler="/checkout",
              status_code=~"5.."
            }[5m]))
            /
            sum(rate(http_requests_total{
              service="order-service",
              handler="/checkout"
            }[5m]))
          )

# Step 3: Burn-rate-based alerts (not arbitrary thresholds)
  - name: slo-alerts
    rules:
      # Fast burn: consuming error budget 14× faster than sustainable
      - alert: CheckoutSLOFastBurn
        expr: |
          job:checkout_request_success:ratio_rate5m < (1 - 14 * 0.001)
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Checkout SLO fast burn — error budget exhausted in < 1 hour"

      # Slow burn: consuming error budget 2× faster than sustainable
      - alert: CheckoutSLOSlowBurn
        expr: |
          job:checkout_request_success:ratio_rate5m < (1 - 2 * 0.001)
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Checkout SLO slow burn — error budget will exhaust within days"
```

### Quick Check

- [ ] SLOs are defined for all user-facing services (written in a document)
- [ ] SLO targets are based on user journey analysis, not guesswork
- [ ] Error budgets are calculated and tracked
- [ ] Alert thresholds are derived from SLO targets (not arbitrary %)
- [ ] Error budget status is visible on the team dashboard

---

## 10. Vendor Lock-in

### Problem

Using a vendor-specific observability SDK directly throughout application code makes it expensive and difficult to switch vendors, compare alternatives, or run locally without vendor agents.

### Example of the Bad Pattern

```python
# Bad: Datadog SDK used directly throughout application code
from ddtrace import tracer   # Datadog-specific import
from ddtrace.contrib.flask import patch_all
patch_all()

# Hundreds of files contain:
with tracer.trace("database.query", service="order-service") as span:
    span.set_tag("db.type", "postgresql")
    span.set_tag("db.statement", query)
    result = db.execute(query)

# Now to switch to Jaeger, Zipkin, or Honeycomb:
# - Rewrite every single trace instrumentation call
# - Change imports in hundreds of files
# - Different APIs, different tagging conventions
# - Major migration project = 2–4 engineer-weeks
```

### Why It's Harmful

- Vendor migration requires touching every instrumented file in the codebase
- Cannot run services locally without vendor agent or vendor API keys
- Vendor pricing increases → cannot switch without massive rewrite cost
- Different vendor APIs between services → inconsistent trace context propagation
- Hard to open-source or share service code that has proprietary SDK calls

### Recommended Fix

```python
# Good: OpenTelemetry as the abstraction layer, vendor as the exporter
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Vendor selection is in configuration, not in application code
def configure_tracing(exporter_backend: str):
    provider = TracerProvider()

    if exporter_backend == "jaeger":
        from opentelemetry.exporter.jaeger.thrift import JaegerExporter
        exporter = JaegerExporter(agent_host_name="jaeger", agent_port=6831)
    elif exporter_backend == "datadog":
        from opentelemetry.exporter.datadog import DatadogExporter
        exporter = DatadogExporter()
    elif exporter_backend == "otlp":
        from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
        exporter = OTLPSpanExporter(endpoint=os.environ["OTLP_ENDPOINT"])
    else:
        from opentelemetry.sdk.trace.export import ConsoleSpanExporter
        exporter = ConsoleSpanExporter()  # Local development

    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)

# Application code is vendor-agnostic:
tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("database.query") as span:
    span.set_attribute("db.system", "postgresql")
    span.set_attribute("db.statement", query)
    result = db.execute(query)
```

### Quick Check

- [ ] All tracing uses OpenTelemetry API, not vendor-specific SDK
- [ ] All metrics use Prometheus client library or OTel metrics API
- [ ] Vendor backend is configured via environment variable or collector config
- [ ] Application code compiles and runs locally without vendor API keys
- [ ] OTel Collector handles vendor routing — not application code

---

## 11. Treating Observability as an Afterthought

### Problem

Services are deployed to production with no instrumentation, then observability is "added later" — but later never comes, or comes only after a major incident proves its absence.

### Example of the Bad Pattern

```yaml
# deployment.yaml — deployed to production
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    spec:
      containers:
        - name: order-service
          image: order-service:1.0.0
          # No livenessProbe — Kubernetes cannot detect hangs
          # No readinessProbe — receives traffic before ready
          # No resource limits — can starve other pods
          # No log configuration — logs going to /dev/null
          # No metrics port — Prometheus cannot scrape it
          # No environment variable for log level
```

```python
# app.py — zero instrumentation
from flask import Flask
app = Flask(__name__)

@app.route("/checkout", methods=["POST"])
def checkout():
    # No logging
    # No metrics
    # No tracing
    # No error handling
    result = process_payment(request.json)
    return jsonify(result)

# When this crashes in production:
# - No logs to understand what happened
# - No metrics to see when it started failing
# - No traces to see the call path
# - No health checks to detect the outage
# Time to detect: user complaint (could be hours)
# Time to diagnose: unknown (no data)
```

### Why It's Harmful

- Mean Time To Detect (MTTD) is measured in hours or user complaints
- MTTR is unbounded — impossible to diagnose without data
- First response to every incident is "add logging" then "redeploy" then "reproduce"
- Production incidents teach the team what observability they need — the most expensive classroom
- SRE/platform team cannot support the service at all

### Recommended Fix

**Observability as part of the Definition of Done:**

```markdown
## Definition of Done — Observability Requirements

Before any service is deployed to production, it MUST have:

### Metrics
- [ ] `/metrics` endpoint exposed and Prometheus scrape configured
- [ ] RED metrics: request rate, error rate, request duration histogram
- [ ] Process metrics: CPU, memory, goroutines/threads
- [ ] At least one business metric (e.g., orders_created_total)

### Logging
- [ ] Structured JSON logging with: timestamp, level, service, correlation_id
- [ ] LOG_LEVEL configurable via environment variable (default: INFO)
- [ ] Log at INFO for all service boundary entry/exit points
- [ ] No sensitive data (passwords, tokens, PII) in logs

### Tracing
- [ ] OpenTelemetry instrumentation with trace context propagation
- [ ] Trace ID injected into all log lines
- [ ] Spans created for all external calls (DB, HTTP, queue)

### Health Checks
- [ ] /livez — liveness probe (process alive)
- [ ] /readyz — readiness probe (dependencies ready)
- [ ] /health — deep health with dependency statuses

### Alerting
- [ ] At minimum: alert on error_rate > 1% for 5 minutes
- [ ] At minimum: alert on p99 latency > SLO target
- [ ] Alerts have runbook_url annotations
- [ ] Alerts routed to team channel
```

### Quick Check

- [ ] Observability requirements are in the team's Definition of Done
- [ ] Pull request template includes observability checklist
- [ ] No service deployed to production without `/metrics`, `/livez`, `/readyz`
- [ ] Pre-production environment validates observability before promoting
- [ ] Observability instrumentation is reviewed in code review (not post-deploy)

---

## 12. Missing Correlation IDs

### Problem

In distributed systems, a single user action triggers requests across multiple services. Without a shared correlation ID, log lines from different services cannot be joined, making distributed debugging rely on fragile timestamp correlation.

### Example of the Bad Pattern

```
# 5 services, zero correlation — try to debug this:

[order-service]   2024-01-15 10:30:00.123 ERROR checkout failed
[payment-service] 2024-01-15 10:30:00.891 ERROR payment declined: invalid card
[inventory-service] 2024-01-15 10:30:00.145 INFO reservation created
[notification-service] 2024-01-15 10:30:01.001 ERROR failed to send email: smtp timeout
[user-service]   2024-01-15 10:30:00.050 INFO user authenticated: id=12345

# Questions impossible to answer:
# - Are these log lines from the same user request or different requests?
# - Did payment decline cause order failure, or was it a different error?
# - Is the email failure related to the checkout failure?
# - The timestamps are within 1 second — coincidence or causation?
```

### Why It's Harmful

- Debugging distributed failures requires examining millions of log lines by timestamp
- Cannot determine causal relationships between log events across services
- Impossible to produce a complete audit trail for a single user action
- Incident investigations require hours of manual log correlation
- Cannot write automated alerts that correlate events across services

### Recommended Fix

```go
// correlation/middleware.go
package correlation

import (
    "context"
    "net/http"
    "github.com/google/uuid"
)

const Header = "X-Correlation-ID"

type contextKey struct{}

func Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := r.Header.Get(Header)
        if id == "" {
            id = uuid.New().String()
        }
        w.Header().Set(Header, id)
        ctx := context.WithValue(r.Context(), contextKey{}, id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func FromContext(ctx context.Context) string {
    if id, ok := ctx.Value(contextKey{}).(string); ok {
        return id
    }
    return ""
}

// Propagate to downstream HTTP clients
func Inject(ctx context.Context, req *http.Request) {
    if id := FromContext(ctx); id != "" {
        req.Header.Set(Header, id)
    }
}
```

**Result — every log line searchable by correlation ID:**

```json
{"service": "order-service",   "correlation_id": "abc-123", "event": "checkout_failed"}
{"service": "payment-service", "correlation_id": "abc-123", "event": "payment_declined", "reason": "invalid_card"}
{"service": "notification-service", "correlation_id": "abc-123", "event": "email_failed"}
```

One `correlation_id=abc-123` query in Loki returns all three — instant causality.

### Quick Check

- [ ] Correlation ID middleware implemented at all service entry points (HTTP, gRPC, queue consumer)
- [ ] Correlation ID propagated in all outgoing HTTP headers
- [ ] Correlation ID propagated in all message queue message headers/properties
- [ ] Correlation ID appears in every structured log line
- [ ] API gateway generates correlation IDs for requests that arrive without one

---

## 13. Using Averages Instead of Percentiles

### Problem

Average latency smooths over the tail of your latency distribution, hiding the experience of the slowest users. A service with 99ms average but 10 second p99 is failing 1% of users — alerting on the average never fires.

### Example of the Bad Pattern

```yaml
# Bad: alerting on average latency
- alert: HighLatency
  expr: avg(http_request_duration_seconds) > 1.0
  for: 5m
  # Problem: avg() is easily dominated by the fast majority
  # Example: 99 requests at 10ms, 1 request at 30,000ms
  # avg = (99 * 10 + 30000) / 100 = 309ms → below 1s threshold, no alert
  # But 1% of users waited 30 seconds!
```

**The math that makes averages misleading:**

```
Requests per minute: 1000
Distribution:
  950 requests → 50ms
  40 requests  → 500ms
  9 requests   → 2,000ms
  1 request    → 60,000ms  ← 1 user waited 60 seconds

avg = (950×50 + 40×500 + 9×2000 + 1×60000) / 1000
    = (47500 + 20000 + 18000 + 60000) / 1000
    = 145.5ms  ← average looks fast

p50  = 50ms    ← half of users
p95  = 500ms   ← 95th percentile is concerning
p99  = 2000ms  ← 10 users/minute experiencing 2s latency
p99.9 = 60000ms ← 1 user/minute experiencing 60s latency
```

### Why It's Harmful

- SLO violations are invisible: 1% of users experiencing 60-second latency goes undetected
- Capacity planning is wrong: average masks the actual peak resource usage
- Performance regressions are hidden: a regression affecting the slow tail doesn't move the average
- On-call engineers are lulled into false confidence by "average looks fine"
- Enterprise customers (who often experience tail latency disproportionately) are impacted without alerting

### Recommended Fix

```yaml
# Good: histogram_quantile for p50/p95/p99 alerting
- alert: HighLatencyP99
  expr: |
    histogram_quantile(0.99,
      sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
    ) > 2.0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "p99 latency > 2s — 1% of users experiencing slow requests"

- alert: HighLatencyP95
  expr: |
    histogram_quantile(0.95,
      sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
    ) > 1.0
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "p95 latency > 1s — 5% of users experiencing slow requests"
```

**Grafana dashboard panel — always show percentiles, not average:**

```
Panel title: "Request Latency"
Queries:
  A: histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) — legend: p50
  B: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) — legend: p95
  C: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) — legend: p99
  D: avg(rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])) — legend: avg (for reference only)
```

### Quick Check

- [ ] All latency alerts use `histogram_quantile` (p95 or p99), not `avg()`
- [ ] Dashboards show p50, p95, p99 latency lines
- [ ] Histogram metrics are configured with appropriate buckets (not defaults)
- [ ] SLO targets are expressed as percentile thresholds, not averages
- [ ] Team documentation explains "we use p99 for SLOs" explicitly

---

## 14. Not Testing Alerts

### Problem

Alert rules are written and deployed without ever being validated. The team discovers the alerts don't fire — or fire incorrectly — during an actual outage. This is the worst possible time to debug your alerting pipeline.

### Example of the Bad Pattern

```yaml
# alerting/rules.yml — deployed 18 months ago, never tested
- alert: DatabaseDown
  expr: up{job="postgresql"} == 0
  for: 5m
  # Problem 1: The `up` metric job label is actually "postgres" not "postgresql"
  #            — this alert will NEVER fire

- alert: HighErrorRate
  expr: error_rate > 0.01
  # Problem 2: `error_rate` metric doesn't exist — it's called `http_error_ratio`
  #            — this alert will NEVER fire

- alert: ServiceRestarting
  expr: increase(kube_pod_container_status_restarts_total[1h]) > 3
  # Problem 3: increase() on a counter with 1h window never resolves during normal operation
  #            — this alert may never resolve (always pending)

# The team finds out about all three problems during a major outage.
```

### Why It's Harmful

- Alert rules with typos or wrong metric names silently never fire
- Alertmanager routing misconfiguration means pages go to the wrong team
- Label mismatches mean severity routing doesn't work as expected
- Teams operate under the false assumption they will be paged — they won't be
- Discovering broken alerts during an outage increases MTTR significantly
- Regulatory/compliance requirements for alerting may be violated

### Recommended Fix

**1. Unit test alert rules with `promtool`:**

```yaml
# tests/alert_tests.yml
rule_files:
  - alerting/order-service-alerts.yml

tests:
  - interval: 1m
    input_series:
      # Simulate high error rate
      - series: 'http_requests_total{status_code="500", service="order-service"}'
        values: '0 10 20 30 40 50 60'
      - series: 'http_requests_total{status_code="200", service="order-service"}'
        values: '0 100 200 300 400 500 600'

    alert_rule_test:
      - eval_time: 10m
        alertname: OrderServiceErrorRateHigh
        exp_alerts:
          - exp_labels:
              severity: critical
              service: order-service
            exp_annotations:
              summary: "Order Service error rate exceeds 1%"
```

```bash
# Run alert unit tests in CI
promtool test rules tests/alert_tests.yml

# Expected output:
# Unit Testing: tests/alert_tests.yml
#   SUCCESS
```

**2. Test Alertmanager routing with `amtool`:**

```bash
# Verify an alert routes to the correct receiver
amtool alert add alertname="OrderServiceErrorRateHigh" \
  severity="critical" \
  service="order-service" \
  --alertmanager.url=http://alertmanager:9093

# Check what receiver it was routed to
amtool config routes test \
  alertname="OrderServiceErrorRateHigh" \
  severity="critical"
# Output: pagerduty-critical

# Test silence matching
amtool silence add alertname="OrderServiceErrorRateHigh" \
  --comment="Testing silence" \
  --duration="5m"
```

**3. Regular alert fire drills:**

```bash
# Simulate an alert firing (inject fake metric via Pushgateway)
echo 'http_error_ratio{service="order-service"} 0.05' | \
  curl --data-binary @- http://pushgateway:9091/metrics/job/fire-drill

# Verify:
# 1. Alert appears in Prometheus alerts page
# 2. Alertmanager receives and routes it
# 3. PagerDuty/Slack notification arrives in < 2 minutes
# 4. Runbook URL is correct and accessible

# Clean up after drill
curl -X DELETE http://pushgateway:9091/metrics/job/fire-drill
```

### Quick Check

- [ ] `promtool test rules` runs in CI pipeline for all alert rule changes
- [ ] Alertmanager routing tested with `amtool config routes test` for each severity level
- [ ] Alert fire drill conducted quarterly (simulate top 3 most critical alerts)
- [ ] All alert rules reference metrics that actually exist (checked with `promtool check rules`)
- [ ] Alert delivery time measured (paged within 2 minutes of condition being met)

---

## Observability Maturity Anti-Patterns

Different anti-patterns are most prevalent at each maturity level. Use this to understand where your organisation sits and what to tackle next.

| Maturity Level | Description | Most Common Anti-Patterns |
|---|---|---|
| **Level 0 — No Observability** | No metrics, no structured logs, no traces, no health checks | #11 (Afterthought), #7 (No structured logging), #6 (Logging too little) |
| **Level 1 — Basic Monitoring** | Some metrics, some logs, no tracing, no SLOs | #14 (Not testing alerts), #8 (Causes not symptoms), #1 (Alert fatigue) |
| **Level 2 — Reactive** | Metrics + logs, alerting exists but noisy, no SLOs, no tracing | #1 (Alert fatigue), #13 (Averages), #9 (No SLOs), #3 (Missing tracing) |
| **Level 3 — Proactive** | Structured logs, metrics, some tracing, starting SLOs | #4 (High cardinality), #12 (No correlation IDs), #2 (Monitoring everything) |
| **Level 4 — Optimised** | SLOs, tracing, error budgets, observability as code | #10 (Vendor lock-in), #5 (Logging too much / cost), Cost optimization |

---

## Quick Reference Checklist

Use this checklist for pre-production reviews and periodic audits.

### Alerting

- [ ] Alerts are symptom-based (error rate, latency, availability) — not cause-based (CPU, memory)
- [ ] All alerts have `severity` labels: critical (pages on-call), warning (chat only)
- [ ] All critical alerts have `runbook_url` annotations
- [ ] All alerts have `for: 5m` (or longer) to prevent flapping
- [ ] Alert thresholds are derived from SLO targets
- [ ] Alert rules are unit-tested with `promtool test rules` in CI
- [ ] Alertmanager routing tested with `amtool config routes test`
- [ ] Quarterly alert fire drill conducted
- [ ] No informational alerts in the primary on-call channel
- [ ] Weekly alert review process in place

### Metrics

- [ ] Total Prometheus active series < 500,000 per instance
- [ ] No label uses `user_id`, `request_id`, `session_id`, or raw URL path
- [ ] Metric names follow snake_case convention with namespace prefix
- [ ] Counters end in `_total`; durations use `_seconds`
- [ ] `metric_relabel_configs` used to drop unused metrics
- [ ] Histogram buckets are customised for service's latency profile
- [ ] Latency alerts use `histogram_quantile()`, not `avg()`
- [ ] `promtool tsdb analyze` run monthly to track cardinality

### Logging

- [ ] All log output is structured JSON
- [ ] `LOG_LEVEL=INFO` in production (not DEBUG)
- [ ] No request/response bodies logged (PII + cost)
- [ ] No HTTP headers logged in full (token leakage risk)
- [ ] Every log line includes `correlation_id` field
- [ ] Every log line includes `service`, `timestamp`, `level` fields
- [ ] Error logs include enough context to reproduce without redeployment
- [ ] Health check and metrics scrape endpoints filtered from log shippers

### Tracing

- [ ] OpenTelemetry used (not vendor-specific SDK)
- [ ] Trace ID injected into all log lines
- [ ] Trace context propagated in all outgoing HTTP headers
- [ ] Trace context propagated in all message queue message headers
- [ ] Sampling rate configured appropriately for traffic volume
- [ ] Tail-based sampling for errors and slow traces

### Health Checks

- [ ] `/livez` implemented (process liveness only)
- [ ] `/readyz` implemented (dependency readiness)
- [ ] `/health` or `/health/detailed` implemented with JSON dependency statuses
- [ ] Kubernetes `livenessProbe` configured
- [ ] Kubernetes `readinessProbe` configured
- [ ] `startupProbe` configured for slow-starting services

### SLOs and Process

- [ ] SLOs defined for all user-facing services (in documentation)
- [ ] Error budgets calculated and tracked
- [ ] Observability requirements in Definition of Done
- [ ] Pull request template includes observability checklist
- [ ] Observability tooling vendor-neutral (OpenTelemetry)
- [ ] Dashboards and alert rules stored in version control

---

## Next Steps

1. **Score yourself** — Use the [Quick Reference Checklist](#quick-reference-checklist) and score your current systems. Any unchecked item is a potential anti-pattern.
2. **Fix high severity first** — Address 🔴 Critical anti-patterns (#1, #4, #8, #9, #11) before medium ones.
3. **Read the companion guide** — [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) describes the correct patterns to replace these anti-patterns.
4. **Define SLOs** — If you have not defined SLOs, [07-SLIS-SLOS-SLAS.md](07-SLIS-SLOS-SLAS.md) is the next document to read.
5. **Instrument with OpenTelemetry** — [04-OPENTELEMETRY.md](04-OPENTELEMETRY.md) covers vendor-neutral instrumentation.
6. **Follow the learning path** — [LEARNING-PATH.md](LEARNING-PATH.md) provides a structured curriculum through all observability topics.

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-01-01 | Initial document — 14 anti-patterns with examples and fixes | Platform Team |
| 1.1 | 2025-01-15 | Added maturity level mapping, quick reference checklist | SRE Team |
| 1.2 | 2025-02-01 | Expanded alert testing section with promtool and amtool examples | Observability Guild |
| 1.3 | 2025-02-15 | Added vendor lock-in anti-pattern, OpenTelemetry fix examples | Platform Team |
