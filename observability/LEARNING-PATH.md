# Observability Learning Path

A structured, self-paced training guide to mastering observability — from foundations and the three pillars to production-grade SLOs, distributed tracing, and anti-pattern elimination. Each phase builds on the previous one, progressing from core concepts to advanced production patterns.

> **Time Estimate:** 10–12 weeks at ~5 hours/week. Adjust pace to your experience level. Engineers with prior Prometheus or logging experience may complete Phases 1–2 in half the time.

---

## How to Use This Guide

1. **Follow the phases in order** — each one builds directly on prior knowledge; jumping ahead creates gaps that slow you down later
2. **Read the linked documents** — they contain the detailed content, code examples, and reference tables
3. **Complete the exercises** — hands-on practice is how observability concepts become intuition; do not skip them
4. **Check yourself** — use the knowledge check items before moving to the next phase; if you cannot answer them confidently, re-read the relevant sections
5. **Build the capstone project** — the final project ties together all six phases and produces a portfolio artifact you can reference in engineering conversations

---

## Phase 1: Foundations (Week 1–2)

### Learning Objectives

- Understand the difference between observability and traditional monitoring
- Learn the three pillars of observability: metrics, logs, and traces — and why all three are needed
- Grasp the Four Golden Signals (Latency, Traffic, Errors, Saturation) and why Google SRE built around them
- Survey the CNCF observability landscape: Prometheus, Grafana, Jaeger, OpenTelemetry, Loki
- Understand when observability investments pay off and how to make the case for them

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [00-OVERVIEW](00-OVERVIEW.md) | What is observability, three pillars, CNCF landscape, observability vs monitoring |
| 2 | [01-METRICS](01-METRICS.md) | Metric types (counter/gauge/histogram/summary), Prometheus data model basics |

### Exercises

**1. Observability Gap Analysis:**

Choose a system you are familiar with — a personal project, a work service, or an open-source application — and perform an observability gap analysis:

- List what you currently know about the system's runtime behaviour (metrics exported, logs available, traces if any)
- For each of the Four Golden Signals, state whether the system currently exposes it:
  - Latency: Is p99 request latency measurable today?
  - Traffic: Can you see requests per second over the past 7 days?
  - Errors: Is the 5xx error rate queryable and alertable?
  - Saturation: Is CPU/memory utilisation tracked?
- Identify the top 3 questions you cannot currently answer about production behaviour
- Estimate the time it would take to diagnose a 50% error rate in this system today, with no additional tooling

Document your findings. You will revisit this analysis after Phase 6.

**2. Local Prometheus Setup:**

Run Prometheus and a sample application locally using Docker Compose:

```yaml
# docker-compose.yml — starter stack
version: "3.8"
services:
  prometheus:
    image: prom/prometheus:v2.48.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  node-exporter:
    image: prom/node-exporter:v1.7.0
    ports:
      - "9100:9100"

  sample-app:
    image: quay.io/brancz/prometheus-example-app:v0.3.0
    ports:
      - "8080:8080"
```

Once running:
- Open the Prometheus UI at `http://localhost:9090`
- Navigate to **Status → Targets** — verify all scrape targets are UP
- Write and execute the following 5 PromQL queries in the Prometheus expression browser:
  1. `up` — what does this tell you?
  2. `rate(http_requests_total[5m])` — what are the current request rates?
  3. `http_request_duration_seconds_bucket` — what buckets are available?
  4. `process_resident_memory_bytes` — how much memory is the sample app using?
  5. A query of your own choice from the available metrics

### Knowledge Check

- [ ] What are the three pillars of observability, and what question does each answer?
- [ ] How is observability different from traditional black-box monitoring?
- [ ] Name the Four Golden Signals and give one real-world example of each
- [ ] What does the Prometheus `up` metric represent, and what is its value when a scrape fails?

---

## Phase 2: Metrics Deep Dive (Week 3–4)

### Learning Objectives

- Master all four Prometheus metric types: Counter, Gauge, Histogram, Summary
- Write fluent PromQL: rate(), increase(), histogram_quantile(), topk(), by/without clauses
- Design and configure Prometheus alerting rules with appropriate severity levels and thresholds
- Build a Grafana dashboard with RED metrics, template variables, and alert annotations
- Understand recording rules and when to use them for performance optimisation

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [01-METRICS](01-METRICS.md) | Full document — all metric types, PromQL, recording rules, federation |
| 2 | [05-ALERTING](05-ALERTING.md) | Prometheus alerting rules, for: clause, labels/annotations, alert grouping |

### Exercises

**1. Instrument a Sample HTTP Server:**

Instrument a Go or Python HTTP server with all four metric types. The server should expose a `/metrics` endpoint that Prometheus can scrape.

*Go (using prometheus/client_golang):*

```go
package main

import (
    "net/http"
    "time"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    // Counter: total requests
    requestsTotal = prometheus.NewCounterVec(prometheus.CounterOpts{
        Name: "myapp_http_requests_total",
        Help: "Total HTTP requests",
    }, []string{"method", "handler", "status_code"})

    // Histogram: request duration
    requestDuration = prometheus.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "myapp_http_request_duration_seconds",
        Help:    "HTTP request duration",
        Buckets: []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10},
    }, []string{"method", "handler"})

    // Gauge: active connections
    activeConnections = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "myapp_active_connections",
        Help: "Current number of active connections",
    })

    // Summary: custom quantiles (use Histogram in production; this is for learning)
    processingTime = prometheus.NewSummary(prometheus.SummaryOpts{
        Name:       "myapp_processing_time_seconds",
        Help:       "Processing time quantiles",
        Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
    })
)
```

After instrumenting, verify:
- All four metric types appear in `http://localhost:8080/metrics`
- Prometheus scrapes the endpoint successfully
- PromQL queries return expected results for each metric type

**2. Build a Grafana Dashboard with RED Metrics:**

Connect Grafana (`http://localhost:3000`) to your Prometheus instance and create a dashboard with:

- **Row 1 — Service Health** (stat panels):
  - Current request rate (RPS)
  - Error rate percentage
  - p99 latency (ms)

- **Row 2 — RED Metrics** (time-series graphs):
  - Request rate over time, broken down by status code
  - Error rate over time (5xx / total)
  - Latency percentiles (p50, p95, p99) on one panel

- **Template variables:**
  - `$handler` — populated from label values
  - `$interval` — options: 1m, 5m, 15m

- **Annotations:**
  - Enable Prometheus alert annotations so fired alerts appear on graphs

Export the dashboard as JSON and save it to your repository.

### Knowledge Check

- [ ] When should you use a Histogram vs a Summary? What are the trade-offs?
- [ ] Why does `avg(http_request_duration_seconds)` give misleading results for latency SLOs?
- [ ] What does `rate(counter[5m])` compute, and why do you need `rate()` on a counter?
- [ ] What is a recording rule, and when does it improve dashboard and alert performance?

---

## Phase 3: Logging and Tracing (Week 5–6)

### Learning Objectives

- Implement structured logging in Go and Python with consistent field conventions
- Understand log levels, retention, and cost-efficient log management
- Set up a log aggregation stack (Loki + Grafana or ELK)
- Add distributed tracing to a multi-service application
- Understand trace structure: spans, parent-child relationships, baggage, context propagation
- Use Jaeger UI to navigate distributed traces and identify bottlenecks

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [02-LOGGING](02-LOGGING.md) | Structured logging, log levels, log aggregation, Loki/ELK, retention |
| 2 | [03-DISTRIBUTED-TRACING](03-DISTRIBUTED-TRACING.md) | Spans, trace context, W3C traceparent, Jaeger, context propagation |

### Exercises

**1. Migrate Unstructured Logs to Structured JSON:**

Take a service with unstructured printf-style logging and migrate it to structured logging with trace ID injection.

Before (bad):

```python
# Bad
import logging
logger = logging.getLogger(__name__)

def process_payment(user_id, amount):
    logger.info(f"Processing payment for user {user_id}, amount ${amount}")
    try:
        result = payment_gateway.charge(user_id, amount)
        logger.info(f"Payment successful: {result.transaction_id}")
    except Exception as e:
        logger.error(f"Payment failed: {e}")
```

After (good) — your task is to rewrite this to:
- Use `structlog` with JSON renderer
- Include: `timestamp`, `level`, `service`, `event`, `user_id`, `correlation_id`, `trace_id`
- Configure log level from `LOG_LEVEL` environment variable
- Add trace ID injection from the current OpenTelemetry span context
- Write at least 3 tests that verify log output contains expected fields

**2. Deploy Jaeger and Instrument Two Services:**

Using Docker Compose, run Jaeger and two simple HTTP services (or use the OpenTelemetry Demo):

```yaml
# Add to your docker-compose.yml
jaeger:
  image: jaegertracing/all-in-one:1.52
  ports:
    - "16686:16686"   # Jaeger UI
    - "4317:4317"     # OTLP gRPC
    - "4318:4318"     # OTLP HTTP
```

Instrument Service A (frontend) to call Service B (backend), propagating trace context:
- Service A: receive a request, create a root span, call Service B with `traceparent` header
- Service B: extract the trace context from the incoming header, create a child span
- Verify in Jaeger UI: the trace shows both services in the same waterfall view

After completing, answer:
- How many spans does a single request produce?
- Where is the time being spent (Service A processing vs Service B)?
- Can you identify the service that is the bottleneck?

### Knowledge Check

- [ ] What fields should every structured log line contain at minimum?
- [ ] Why is it dangerous to log HTTP headers or request/response bodies in production?
- [ ] What is the W3C `traceparent` header format and what does each segment encode?
- [ ] What is the difference between a root span and a child span?

---

## Phase 4: OpenTelemetry and APM (Week 7–8)

### Learning Objectives

- Understand the OpenTelemetry project: API, SDK, Collector architecture
- Implement OTel auto-instrumentation and manual instrumentation in at least one language
- Configure the OpenTelemetry Collector with multiple exporters (Jaeger + Prometheus + logging)
- Understand how OTel unifies metrics, logs, and traces under a single SDK
- Evaluate APM tools (Datadog, New Relic, Dynatrace, Grafana Cloud) and when managed APM makes sense
- Understand the difference between open-source self-hosted and vendor-managed observability

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [04-OPENTELEMETRY](04-OPENTELEMETRY.md) | OTel API/SDK/Collector architecture, auto-instrumentation, context propagation |
| 2 | [06-APM](06-APM.md) | APM tool comparison, managed vs self-hosted, cost model, selection criteria |

### Exercises

**1. Auto-Instrument a Java or Node.js Application:**

Choose Java (Spring Boot) or Node.js (Express) and apply OpenTelemetry auto-instrumentation:

*Node.js example — zero-code instrumentation:*

```bash
npm install @opentelemetry/auto-instrumentations-node \
            @opentelemetry/sdk-node \
            @opentelemetry/exporter-trace-otlp-http \
            @opentelemetry/exporter-prometheus
```

```javascript
// tracing.js — loaded before application code
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');
const { PrometheusExporter } = require('@opentelemetry/exporter-prometheus');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://jaeger:4318/v1/traces',
  }),
  metricReader: new PrometheusExporter({ port: 9464 }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

```bash
# Start app with auto-instrumentation
node --require ./tracing.js app.js
```

Complete the exercise by:
- Sending test requests to the instrumented app
- Verifying traces appear in Jaeger
- Verifying metrics appear at `http://localhost:9464/metrics`
- Identifying which auto-instrumentation libraries are active (HTTP, database, etc.)

**2. Configure OTel Collector with Multiple Exporters:**

Write an OTel Collector configuration that:
- Receives traces and metrics via OTLP (gRPC and HTTP)
- Exports traces to Jaeger
- Exports metrics to Prometheus (via Prometheus Remote Write or scrape endpoint)
- Exports all telemetry to the logging exporter (for debugging)
- Applies a `batch` processor and a `memory_limiter` resource processor

```yaml
# otel-collector-config.yaml — your task: fill in the exporters section
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:
    limit_mib: 512
    check_interval: 1s
  batch:
    send_batch_size: 1024
    timeout: 1s

exporters:
  # TODO: Add jaeger, prometheus, and logging exporters here

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [jaeger, logging]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus, logging]
```

Test your configuration with `otelcol validate --config=otel-collector-config.yaml`.

### Knowledge Check

- [ ] What is the difference between the OTel API and the OTel SDK?
- [ ] Why is the OTel Collector preferred over exporting directly from the application to Jaeger or Prometheus?
- [ ] What is auto-instrumentation and what does it typically instrument automatically (HTTP, database, etc.)?
- [ ] When would you choose a managed APM (Datadog, New Relic) over self-hosted Prometheus + Jaeger?

---

## Phase 5: Alerting and SLOs (Week 9–10)

### Learning Objectives

- Design meaningful, actionable alerts based on symptoms rather than causes
- Configure Alertmanager with routing, receivers, inhibition, and silences
- Define Service Level Indicators (SLIs) and Service Level Objectives (SLOs)
- Calculate error budgets and understand their role in prioritising reliability work
- Implement multi-window, multi-burn-rate alerting for SLO compliance
- Understand SLA vs SLO vs SLI distinctions

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [05-ALERTING](05-ALERTING.md) | Full document — Prometheus rules, Alertmanager config, routing, PagerDuty integration |
| 2 | [07-SLIS-SLOS-SLAS](07-SLIS-SLOS-SLAS.md) | Full document — SLI definitions, SLO targets, error budgets, burn-rate alerting |

### Exercises

**1. Design Five Alerting Rules for a Web API:**

Design five alerting rules for a checkout API service, following symptom-based alerting principles. For each rule, write the complete Prometheus rule YAML including:
- Meaningful `alert` name
- PromQL `expr` using the correct metric names from a typical Prometheus instrumented service
- Appropriate `for` duration
- `severity` label: `critical` or `warning`
- `summary` and `description` annotations
- `runbook_url` annotation

The five alerts should cover:
1. High error rate (5xx errors > 1%)
2. High p99 latency (> 2 seconds)
3. Service completely down (no successful requests for 2 minutes)
4. Error budget burn rate (fast burn: 14× for 2 minutes)
5. Pod restarting repeatedly (crash loop indicator)

Then write a complete `alertmanager.yml` that:
- Routes critical alerts to PagerDuty
- Routes warning alerts to Slack `#platform-alerts`
- Groups alerts by `service` and `alertname`
- Applies a 30-second `group_wait` and 5-minute `group_interval`
- Inhibits warning alerts when a critical alert for the same service fires

**2. Define SLOs and Build an Error Budget Dashboard:**

Define SLOs for a sample checkout service with the following requirements:
- The service processes 10,000 orders per day
- Business requirement: "We must successfully process at least 99.5% of checkout attempts"
- Latency requirement: "95% of checkout requests must complete within 3 seconds"

Tasks:
- Write the SLO definition document (can be markdown): service name, SLI definition, SLO target, measurement window
- Calculate the monthly error budget: how many failed requests are allowed per month?
- Implement Prometheus recording rules to track the SLI over rolling 5-minute, 1-hour, and 6-hour windows
- Create an error budget dashboard in Grafana with:
  - Current SLO compliance (%)
  - Error budget remaining (% and absolute count)
  - Burn rate over the past 24 hours
  - Projected time until error budget exhaustion at current burn rate

### Knowledge Check

- [ ] What is the difference between a cause-based alert (CPU > 80%) and a symptom-based alert?
- [ ] What is an error budget, and how does it change team behaviour around deployments?
- [ ] Why is multi-burn-rate alerting (fast burn + slow burn) better than a single threshold alert for SLOs?
- [ ] What is the difference between an SLI, an SLO, and an SLA?

---

## Phase 6: Production Observability (Week 11–12)

### Learning Objectives

- Apply observability best practices across metrics, logging, tracing, and alerting
- Identify and remediate all 14 anti-patterns from the anti-patterns document
- Build a production-readiness checklist for observability
- Understand cost optimisation strategies for observability infrastructure
- Implement health check patterns (liveness, readiness, startup, deep)
- Understand observability as code: dashboards in Grafonnet, alerts in Terraform

### Reading

| Order | Document | Focus Areas |
|-------|----------|-------------|
| 1 | [08-BEST-PRACTICES](08-BEST-PRACTICES.md) | Full document — instrumentation standards, cardinality, health checks, dashboard design |
| 2 | [09-ANTI-PATTERNS](09-ANTI-PATTERNS.md) | Full document — all 14 anti-patterns, quick reference checklist |

### Exercises

**1. Observability Best Practices Audit:**

Select a service (your own, the capstone, or an open-source project) and perform a full observability audit against the best practices checklist from [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md):

Complete the following checklist and document findings for each item:

```
Metrics:
  [ ] RED metrics exposed (rate, errors, duration histogram)
  [ ] Metric names follow convention (snake_case, namespace, unit suffix)
  [ ] No high-cardinality labels (user_id, request_id, raw URL)
  [ ] Histogram buckets appropriate for service latency profile
  [ ] Total active series < 500,000

Logging:
  [ ] Structured JSON output
  [ ] Log level configurable via environment variable
  [ ] Correlation ID in every log line
  [ ] No PII or credentials in logs
  [ ] Health check and metrics endpoint logs filtered from shipper

Tracing:
  [ ] OpenTelemetry (not vendor-specific) SDK
  [ ] Trace ID injected into log lines
  [ ] Context propagated in all outgoing HTTP and queue calls
  [ ] Sampling configured appropriately

Alerting:
  [ ] Alerts are symptom-based
  [ ] Alert rules unit-tested with promtool
  [ ] All critical alerts have runbook_url
  [ ] SLOs defined and error budgets tracked

Health Checks:
  [ ] /livez (liveness)
  [ ] /readyz (readiness)
  [ ] /health (deep check with dependency statuses)
```

For each unchecked item, write a one-paragraph remediation plan: what to change, estimated effort, and priority.

**2. Anti-Pattern Identification Exercise:**

Review the following system description and identify at least 5 anti-patterns from [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md):

> "We run a payments platform with 8 microservices. Our Prometheus instance has 2.1 million active time series — we're not sure what most of them are. We log everything at DEBUG level in production because 'it helps with debugging'. Logs are plain-text with printf-style formatting. We have 180 alerts configured; our on-call team acknowledges about 160 of them without looking because they always fire. The remaining 20 alerts are the ones we actually care about, but they've never been tested. Our latency alert fires when `avg(response_time) > 500ms` — it fired last week but turned out the service was actually fine. We don't have SLOs. Each service uses a different tracing SDK (some use Datadog, some use Zipkin, some have nothing). We have no health check endpoints."

For each anti-pattern identified:
- State the anti-pattern name and number
- Quote the specific evidence from the description
- Write the recommended fix in 2–3 sentences

<details>
<summary>Anti-patterns identified (click to reveal suggested answers)</summary>

1. **#2 Monitoring Everything** — 2.1 million active time series; team does not know what most are
2. **#5 Logging Too Much** — DEBUG level in production
3. **#7 Missing Structured Logging** — printf-style plain-text logs
4. **#1 Alert Fatigue** — 160/180 alerts acknowledged without reading; 20 actually meaningful alerts buried in noise
5. **#14 Not Testing Alerts** — the 20 meaningful alerts have never been tested
6. **#13 Using Averages Instead of Percentiles** — latency alert on `avg(response_time)`
7. **#9 Not Having SLOs** — "We don't have SLOs"
8. **#10 Vendor Lock-in** — mixed tracing SDKs (Datadog, Zipkin, nothing) across services
9. **#11 Treating Observability as an Afterthought** — some services have no tracing; no health checks
10. **#8 Monitoring Causes Instead of Symptoms** — (implied by no SLOs and arbitrary thresholds)

</details>

### Knowledge Check

- [ ] What are the four categories from the best practices checklist (Metrics, Logging, Tracing, Alerting) and name two rules from each?
- [ ] What is cardinality in the context of Prometheus, and what is the impact of a cardinality explosion?
- [ ] What is the difference between a liveness probe and a readiness probe in Kubernetes?
- [ ] What does "observability as code" mean, and what are two benefits it provides?

---

## Capstone Project: Instrument the OpenTelemetry Demo

Design and implement production-grade observability for the **OpenTelemetry Demo** — a microservices e-commerce application maintained by the OpenTelemetry project that includes services in Go, Python, Java, Node.js, and .NET.

Repository: `https://github.com/open-telemetry/opentelemetry-demo`

### System Architecture

```
                    ┌─────────────────────┐
                    │    Load Generator   │
                    │     (Locust)        │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │    Frontend         │
                    │    (Next.js)        │
                    └──────┬──────┬───────┘
                           │      │
              ┌────────────▼┐    ┌▼────────────┐
              │  Frontend   │    │  Ad Service  │
              │  Proxy      │    │  (Java)      │
              │  (Envoy)    │    └─────────────┘
              └─────┬───────┘
        ┌───────────┼──────────────┐
        │           │              │
   ┌────▼────┐ ┌────▼────┐  ┌─────▼────┐
   │Product  │ │Cart     │  │Checkout  │
   │Catalog  │ │Service  │  │Service   │
   │(Go)     │ │(DotNET) │  │(Go)      │
   └─────────┘ └─────────┘  └────┬─────┘
                                  │
              ┌───────────────────┼───────────┐
              │                   │           │
        ┌─────▼────┐        ┌─────▼──┐  ┌────▼─────┐
        │Payment   │        │Shipping│  │Email     │
        │Service   │        │Service │  │Service   │
        │(Node.js) │        │(Rust)  │  │(Python)  │
        └──────────┘        └────────┘  └──────────┘
              │
        ┌─────▼────┐
        │Currency  │
        │Service   │
        │(Node.js) │
        └──────────┘
```

### Requirements

| Requirement | What to Implement |
|-------------|-------------------|
| **Metrics** | Verify RED metrics are exposed for all services; add custom business metrics (orders_placed_total, cart_abandonment_ratio) |
| **Structured logging** | Ensure trace ID is injected into log lines for at least 3 services |
| **Distributed tracing** | Verify end-to-end trace from Load Generator through Checkout to Payment; identify the service with the highest p99 latency |
| **SLOs** | Define SLOs for Checkout Service: 99.9% success rate, p99 < 2s; implement recording rules |
| **Alerting** | Write 5 alerting rules based on symptoms; configure Alertmanager routing to Slack |
| **Dashboard** | Build a Grafana dashboard with: SLO health, RED metrics for top 3 services, error budget remaining |
| **Health checks** | Verify /livez and /readyz are implemented; add deep health check to one service |
| **Anti-patterns** | Run the quick reference checklist from 09-ANTI-PATTERNS.md; document and fix at least 3 issues found |
| **OTel Collector** | Configure a multi-pipeline Collector: traces to Jaeger, metrics to Prometheus, logs to stdout |
| **Sampling** | Configure tail-based sampling: 100% for errors and slow traces, 5% for everything else |

### Evaluation Criteria

| Area | What to Verify |
|------|----------------|
| **Foundations** | Can you answer "Is the checkout service healthy right now?" from a dashboard in < 10 seconds? |
| **RED Metrics** | Are request rate, error rate, and p99 latency visible for all 3 primary services? |
| **Tracing** | Can you find the full trace for a failed checkout request in Jaeger? |
| **SLOs** | Is the error budget dashboard showing the current burn rate and remaining budget? |
| **Alerting** | Do all 5 alert rules pass `promtool test rules`? Does a simulated failure trigger the correct alert? |
| **Anti-patterns** | Are all 14 anti-patterns from the checklist evaluated? Are findings documented? |
| **Documentation** | Is there a README explaining the observability stack, how to run it, and how to use the dashboards? |

---

## Quick Reference: Document Map

| # | Document | Phase | Key Topics |
|---|----------|-------|------------|
| 00 | [OVERVIEW](00-OVERVIEW.md) | 1 | Three pillars, observability vs monitoring, CNCF landscape |
| 01 | [METRICS](01-METRICS.md) | 1, 2 | Counter/Gauge/Histogram/Summary, PromQL, recording rules |
| 02 | [LOGGING](02-LOGGING.md) | 3 | Structured logging, log levels, Loki, ELK, retention |
| 03 | [DISTRIBUTED-TRACING](03-DISTRIBUTED-TRACING.md) | 3 | Spans, context propagation, Jaeger, Zipkin, W3C traceparent |
| 04 | [OPENTELEMETRY](04-OPENTELEMETRY.md) | 4 | OTel API/SDK/Collector, auto-instrumentation, multi-language |
| 05 | [ALERTING](05-ALERTING.md) | 2, 5 | Prometheus rules, Alertmanager, PagerDuty, symptom-based alerting |
| 06 | [APM](06-APM.md) | 4 | Datadog, New Relic, Dynatrace, Grafana Cloud, build vs buy |
| 07 | [SLIS-SLOS-SLAS](07-SLIS-SLOS-SLAS.md) | 5 | SLI/SLO/SLA definitions, error budgets, burn-rate alerting |
| 08 | [BEST-PRACTICES](08-BEST-PRACTICES.md) | 6 | Golden signals, RED, USE, cardinality, health checks, dashboards |
| 09 | [ANTI-PATTERNS](09-ANTI-PATTERNS.md) | 6 | 14 anti-patterns, quick reference checklist, maturity levels |
| — | [LEARNING-PATH](LEARNING-PATH.md) | All | This document — structured 6-phase curriculum |

---

## Recommended Resources

### Books

| Book | Author | Focus |
|------|--------|-------|
| *Site Reliability Engineering* | Google SRE Team | Golden signals, error budgets, SLOs — the origin of modern observability |
| *The Site Reliability Workbook* | Google SRE Team | Practical SLO implementation with worked examples |
| *Observability Engineering* | Charity Majors, Liz Fong-Jones, George Miranda | Modern observability, high-cardinality events, OpenTelemetry |
| *Distributed Systems Observability* | Cindy Sridharan | Three pillars deep dive, tracing architecture |
| *Prometheus: Up & Running* | Brian Brazil | Comprehensive Prometheus, PromQL, alerting reference |
| *Cloud Native Patterns* | Cornelia Davis | Twelve-factor observability, cloud-native instrumentation |

### Online Resources

- **opentelemetry.io** — Official OTel documentation, getting-started guides, language SDKs
- **prometheus.io/docs** — Prometheus documentation, PromQL reference, best practices
- **grafana.com/docs** — Grafana dashboarding, Loki, Tempo, Mimir documentation
- **sre.google/sre-book** — Free online SRE book (Chapters 6, 10, 12–13 most relevant)
- **sre.google/workbook** — Free online SRE workbook (Chapter 5: SLOs in Practice)
- **last9.io/blog** — Practical observability articles on cardinality and cost
- **brendangregg.com** — USE method, Linux performance tools, flame graphs

### Tools

| Tool | Purpose | Phase |
|------|---------|-------|
| Prometheus | Metrics collection and alerting | 1, 2, 5 |
| Grafana | Dashboards and visualisation | 2, 5, 6 |
| Jaeger / Grafana Tempo | Distributed trace storage and UI | 3, 4 |
| Loki | Log aggregation (Grafana ecosystem) | 3 |
| OpenTelemetry Collector | Telemetry pipeline and routing | 4 |
| Alertmanager | Alert routing, grouping, silencing | 5 |
| promtool | Prometheus rule testing and linting | 5, 6 |
| amtool | Alertmanager testing and inspection | 5, 6 |
| k6 | Load testing to generate observable traffic | 2, 5 |
| Locust | Load testing (Python, used by OTel Demo) | Capstone |
| Docker Compose | Running local observability stacks | 1–4 |
| Kubernetes (kind/minikube) | Running production-like observability | 5–6, Capstone |
