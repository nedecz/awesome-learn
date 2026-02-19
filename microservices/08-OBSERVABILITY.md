# Microservices Observability

## Table of Contents

1. [Overview](#overview)
2. [Three Pillars of Observability](#three-pillars-of-observability)
3. [Distributed Tracing](#distributed-tracing)
4. [Centralized Logging](#centralized-logging)
5. [Metrics and Monitoring](#metrics-and-monitoring)
6. [Health Checks](#health-checks)
7. [Alerting Strategies](#alerting-strategies)
8. [OpenTelemetry](#opentelemetry)
9. [Best Practices](#best-practices)
10. [Next Steps](#next-steps)

## Overview

Observability is the ability to understand the internal state of a system by examining its external outputs. In microservices, where requests traverse multiple services, observability is essential for debugging, performance tuning, and incident response.

### Target Audience

- SREs building monitoring and alerting infrastructure
- Developers instrumenting their microservices
- Platform engineers selecting and deploying observability tools

### Scope

- Distributed tracing across service boundaries
- Centralized logging with correlation
- Metrics collection and dashboard design
- Health check patterns and readiness probes
- Alerting strategies and SLO-based monitoring
- OpenTelemetry as a unified observability framework

## Three Pillars of Observability

```
┌──────────────────────────────────────────────────────────────┐
│                    Observability                              │
│                                                               │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │
│  │    Traces       │  │     Logs       │  │    Metrics     │ │
│  │                 │  │                │  │                │ │
│  │ Request flow   │  │ Event details  │  │ Aggregated     │ │
│  │ across services│  │ per service    │  │ measurements   │ │
│  │                 │  │                │  │                │ │
│  │ "What path did │  │ "What happened │  │ "How is the    │ │
│  │  this request  │  │  at this point │  │  system        │ │
│  │  take?"        │  │  in time?"     │  │  performing?"  │ │
│  └────────────────┘  └────────────────┘  └────────────────┘ │
│                                                               │
│  High cardinality     Unstructured/        Low cardinality    │
│  Per-request detail   structured text      Aggregated trends  │
└──────────────────────────────────────────────────────────────┘
```

| Pillar | Purpose | Examples |
|---|---|---|
| **Traces** | Track a request across multiple services | Jaeger, Zipkin, AWS X-Ray |
| **Logs** | Capture detailed events within a service | ELK Stack, Loki, CloudWatch Logs |
| **Metrics** | Measure system behavior over time | Prometheus, Datadog, CloudWatch Metrics |

## Distributed Tracing

Distributed tracing tracks a single request as it flows through multiple services, providing visibility into latency, errors, and dependencies.

### How It Works

```
Client Request: POST /orders
    │
    │ trace-id: abc123
    │ span-id: span-1
    ▼
┌──────────────┐    trace-id: abc123    ┌──────────────┐
│ API Gateway  │ ──────────────────────▶│ Order Service│
│ span: span-1 │    span-id: span-2     │ span: span-2 │
│ 0ms          │                         │ 5ms           │
└──────────────┘                         └──────┬───────┘
                                                │
                          trace-id: abc123      │
                          span-id: span-3       │
                                                ▼
                                        ┌──────────────┐
                                        │Payment Service│
                                        │ span: span-3 │
                                        │ 50ms          │
                                        └──────┬───────┘
                                               │
                         trace-id: abc123      │
                         span-id: span-4       │
                                               ▼
                                       ┌───────────────┐
                                       │Inventory Service│
                                       │ span: span-4  │
                                       │ 20ms           │
                                       └───────────────┘

Trace View:
├── API Gateway (0–80ms)
│   └── Order Service (5–75ms)
│       ├── Payment Service (10–60ms)
│       └── Inventory Service (60–80ms)
```

### Context Propagation

Trace context must be propagated across service boundaries via HTTP headers.

| Standard | Header | Example |
|---|---|---|
| **W3C Trace Context** | `traceparent` | `00-abc123-span1-01` |
| **B3 (Zipkin)** | `X-B3-TraceId`, `X-B3-SpanId` | `X-B3-TraceId: abc123` |
| **Jaeger** | `uber-trace-id` | `abc123:span1:0:1` |

### Tracing Tools

| Tool | Deployment | Key Features |
|---|---|---|
| **Jaeger** | Self-hosted | CNCF graduated, Kubernetes-native, adaptive sampling |
| **Zipkin** | Self-hosted | Lightweight, language-agnostic, community-driven |
| **AWS X-Ray** | Managed | AWS-native, Lambda support, service map |
| **Grafana Tempo** | Self-hosted/Cloud | Integrates with Grafana, object storage backend |

## Centralized Logging

In microservices, logs from dozens of services must be aggregated into a central location for search and analysis.

### Logging Architecture

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Service A│  │ Service B│  │ Service C│
│  (logs)  │  │  (logs)  │  │  (logs)  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │              │
     └──────┬──────┴──────┬───────┘
            │             │
     ┌──────▼──────┐ ┌───▼─────────┐
     │ Log Shipper │ │ Log Shipper │  (Fluentd, Filebeat,
     │ (sidecar)   │ │ (DaemonSet) │   Fluent Bit)
     └──────┬──────┘ └──────┬──────┘
            │               │
     ┌──────▼───────────────▼──────┐
     │    Log Aggregation          │
     │    (Elasticsearch, Loki)    │
     └──────────────┬──────────────┘
                    │
             ┌──────▼──────┐
             │  Dashboard  │
             │  (Kibana,   │
             │   Grafana)  │
             └─────────────┘
```

### Structured Logging

Always use structured (JSON) logs. They are machine-parseable and enable powerful querying.

```json
{
  "timestamp": "2025-01-15T10:30:00.123Z",
  "level": "INFO",
  "service": "order-service",
  "traceId": "abc123",
  "spanId": "span-2",
  "message": "Order created successfully",
  "orderId": "order-789",
  "customerId": "cust-456",
  "totalAmount": 99.99,
  "duration_ms": 45
}
```

### Correlation IDs

Every request entering the system gets a unique correlation ID that is propagated through all services. This enables tracing a single request across all log entries.

```
Client → API Gateway (generates correlationId: req-xyz)
  → Order Service (logs with correlationId: req-xyz)
    → Payment Service (logs with correlationId: req-xyz)
    → Inventory Service (logs with correlationId: req-xyz)
```

### Logging Tools

| Tool | Type | Key Features |
|---|---|---|
| **ELK Stack** | Self-hosted | Elasticsearch + Logstash + Kibana; powerful full-text search |
| **Grafana Loki** | Self-hosted/Cloud | Label-based indexing, low resource usage, Grafana-native |
| **AWS CloudWatch Logs** | Managed | AWS-native, log groups, metric filters, log insights |
| **Datadog Logs** | SaaS | Unified platform, log patterns, anomaly detection |

## Metrics and Monitoring

Metrics are numeric measurements collected at regular intervals. They are the foundation for dashboards, alerting, and capacity planning.

### Metric Types

| Type | Description | Example |
|---|---|---|
| **Counter** | Monotonically increasing value | Total requests served, total errors |
| **Gauge** | Value that can go up or down | Current queue depth, active connections |
| **Histogram** | Distribution of values in buckets | Request latency distribution |
| **Summary** | Pre-computed quantiles | P50, P95, P99 latency |

### RED Method (for request-driven services)

| Metric | What to Measure |
|---|---|
| **R**ate | Requests per second |
| **E**rrors | Error rate (failed requests / total requests) |
| **D**uration | Latency distribution (P50, P95, P99) |

### USE Method (for infrastructure resources)

| Metric | What to Measure |
|---|---|
| **U**tilization | How busy the resource is (CPU %, memory %) |
| **S**aturation | How much work is queued (queue depth) |
| **E**rrors | Error count for the resource |

### Monitoring Tools

| Tool | Type | Key Features |
|---|---|---|
| **Prometheus** | Self-hosted | Pull-based, PromQL, Kubernetes-native, CNCF graduated |
| **Grafana** | Self-hosted/Cloud | Dashboard visualization, supports multiple data sources |
| **Datadog** | SaaS | Unified metrics, traces, logs; auto-discovery |
| **AWS CloudWatch** | Managed | AWS-native, custom metrics, alarms |

## Health Checks

### Health Check Endpoints

Every service should expose health check endpoints that orchestrators and load balancers use to determine service availability.

```
GET /health/live     → Is the process alive?
GET /health/ready    → Can it accept requests?
GET /health/startup  → Has initialization completed?
```

### Health Check Response

```json
{
  "status": "UP",
  "checks": [
    { "name": "database", "status": "UP", "duration_ms": 5 },
    { "name": "redis", "status": "UP", "duration_ms": 1 },
    { "name": "kafka", "status": "UP", "duration_ms": 3 }
  ]
}
```

## Alerting Strategies

### SLO-Based Alerting

Define Service Level Objectives (SLOs) and alert when the error budget is being consumed too quickly.

```
SLI (Service Level Indicator):
  - Percentage of requests completing in < 500ms

SLO (Service Level Objective):
  - 99.9% of requests complete in < 500ms over 30 days

Error Budget:
  - 0.1% of requests can be slow = ~43 minutes of downtime/month

Alert:
  - If error budget burn rate exceeds 10x normal → page on-call
  - If error budget burn rate exceeds 2x normal → create a ticket
```

### Alert Best Practices

- ✅ Alert on symptoms (user-facing impact), not causes
- ✅ Use multi-window, multi-burn-rate alerts to reduce noise
- ✅ Include runbook links in every alert
- ✅ Set different severity levels (page vs. ticket vs. informational)
- ❌ Do not alert on every metric threshold — alert fatigue kills response quality
- ❌ Do not alert on things that auto-heal (e.g., single Pod restart)

## OpenTelemetry

OpenTelemetry (OTel) is a vendor-neutral, open-source observability framework that provides unified APIs, SDKs, and tools for traces, metrics, and logs.

### Architecture

```
┌──────────────────────────────────────────────────────┐
│                  Application                          │
│  ┌────────────────────────────────────────────────┐  │
│  │  OpenTelemetry SDK                              │  │
│  │  - Traces  - Metrics  - Logs                   │  │
│  └───────────────────┬────────────────────────────┘  │
└──────────────────────┼───────────────────────────────┘
                       │ OTLP (OpenTelemetry Protocol)
                       ▼
              ┌────────────────┐
              │  OTel Collector │
              │                 │
              │ Receives        │
              │ Processes       │
              │ Exports         │
              └──┬──────┬───┬──┘
                 │      │   │
          ┌──────┘      │   └──────┐
          ▼             ▼          ▼
   ┌───────────┐ ┌──────────┐ ┌──────────┐
   │  Jaeger   │ │Prometheus│ │   Loki   │
   │ (traces)  │ │(metrics) │ │ (logs)   │
   └───────────┘ └──────────┘ └──────────┘
```

### Why OpenTelemetry

- **Vendor-neutral** — Switch backends without changing instrumentation
- **Unified** — Single SDK for traces, metrics, and logs
- **Auto-instrumentation** — Automatic instrumentation for popular frameworks
- **CNCF project** — Wide community support and adoption

## Best Practices

### Tracing

- ✅ Propagate trace context (W3C Trace Context) across all service boundaries
- ✅ Add meaningful span attributes (user ID, order ID, error type)
- ✅ Use sampling in production to manage volume (head-based or tail-based)
- ❌ Do not log sensitive data in trace attributes

### Logging

- ✅ Use structured logging (JSON) everywhere
- ✅ Include trace ID and span ID in every log entry
- ✅ Log at appropriate levels (ERROR for failures, INFO for business events, DEBUG for development)
- ❌ Do not log sensitive data (passwords, tokens, PII)
- ❌ Do not use `print` or `console.log` in production services

### Metrics

- ✅ Instrument every service with RED metrics (Rate, Errors, Duration)
- ✅ Use histograms for latency to capture the full distribution
- ✅ Add labels for service, endpoint, status code, and environment
- ❌ Do not create high-cardinality labels (e.g., user ID as a label)

## Next Steps

Continue to [Deployment Strategies](09-DEPLOYMENT-STRATEGIES.md) to learn about blue-green deployments, canary releases, rolling updates, feature flags, and GitOps practices for microservices.
