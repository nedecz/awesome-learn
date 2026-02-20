# Observability Learning Resources

A comprehensive guide to building observable systems — from the three pillars (metrics, logs, traces) and the Four Golden Signals to SLOs, OpenTelemetry, APM, and production best practices.

## 📚 Documentation Structure

| Document | Description | When to Read |
|----------|-------------|--------------|
| [00-OVERVIEW](00-OVERVIEW.md) | What is observability, three pillars, CNCF landscape, observability vs monitoring | **Start here** |
| [01-METRICS](01-METRICS.md) | Prometheus metric types, PromQL, recording rules, federation | When instrumenting or querying metrics |
| [02-LOGGING](02-LOGGING.md) | Structured logging, log levels, Loki, ELK stack, log retention | When building or improving logging |
| [03-DISTRIBUTED-TRACING](03-DISTRIBUTED-TRACING.md) | Spans, trace context, W3C traceparent, Jaeger, Zipkin | When adding tracing to distributed services |
| [04-OPENTELEMETRY](04-OPENTELEMETRY.md) | OTel API/SDK/Collector, auto-instrumentation, multi-language examples | When standardising instrumentation |
| [05-ALERTING](05-ALERTING.md) | Prometheus alerting rules, Alertmanager, routing, PagerDuty/OpsGenie | When designing or improving alerting |
| [06-APM](06-APM.md) | Datadog, New Relic, Dynatrace, Grafana Cloud — APM tool evaluation | When evaluating managed observability |
| [07-SLIS-SLOS-SLAS](07-SLIS-SLOS-SLAS.md) | SLI/SLO/SLA definitions, error budgets, burn-rate alerting | When defining service reliability targets |
| [08-BEST-PRACTICES](08-BEST-PRACTICES.md) | Four Golden Signals, RED/USE methods, cardinality, health checks, dashboards | **Essential — production checklist** |
| [09-ANTI-PATTERNS](09-ANTI-PATTERNS.md) | 14 common observability mistakes and how to fix them | **Essential — what NOT to do** |
| [LEARNING-PATH](LEARNING-PATH.md) | Structured 10–12 week training guide with exercises and knowledge checks | **Start here** after the Overview |
| [README](README.md) | This index file | Start here for navigation |

## 🚀 Quick Start

### For Beginners

1. **Read the Overview** ([00-OVERVIEW](00-OVERVIEW.md))
   - Understand the three pillars: metrics, logs, traces
   - Learn the difference between monitoring and observability
   - Survey the CNCF observability landscape

2. **Set Up a Local Observability Stack** ([01-METRICS](01-METRICS.md))
   - Run Prometheus + Grafana with Docker Compose
   - Scrape a sample application and explore metrics
   - Write your first PromQL queries

3. **Instrument Your First Application** ([04-OPENTELEMETRY](04-OPENTELEMETRY.md))
   - Apply OpenTelemetry auto-instrumentation
   - Export traces to Jaeger and metrics to Prometheus
   - Inject trace IDs into structured log output

4. **Follow the Learning Path** ([LEARNING-PATH](LEARNING-PATH.md))
   - Structured 6-phase curriculum over 10–12 weeks
   - Hands-on exercises, knowledge checks, and a capstone project

### For Experienced Users

1. **Review Best Practices** ([08-BEST-PRACTICES](08-BEST-PRACTICES.md))
   - Four Golden Signals and RED/USE methods
   - Instrumentation standards and cardinality management
   - Health check patterns and dashboard design principles

2. **Audit Against Anti-Patterns** ([09-ANTI-PATTERNS](09-ANTI-PATTERNS.md))
   - 14 common observability mistakes with code-level examples
   - Quick reference checklist for pre-production reviews

3. **Standardise with OpenTelemetry** ([04-OPENTELEMETRY](04-OPENTELEMETRY.md))
   - Replace vendor-specific SDKs with OTel
   - Configure the OTel Collector for multi-backend routing

4. **Define SLOs and Error Budgets** ([07-SLIS-SLOS-SLAS](07-SLIS-SLOS-SLAS.md))
   - Move from arbitrary thresholds to SLO-based alerting
   - Implement burn-rate alerting and error budget dashboards

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Applications / Services                       │
│   (Go, Python, Java, Node.js, .NET — any language)             │
└────────────────────────┬────────────────────────────────────────┘
                         │ emit telemetry
┌────────────────────────▼────────────────────────────────────────┐
│                  Instrumentation Layer                           │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │              OpenTelemetry SDK                           │  │
│   │   Metrics API │ Tracing API │ Logging API                │  │
│   └──────────────────────┬───────────────────────────────────┘  │
└──────────────────────────│──────────────────────────────────────┘
                           │ OTLP (gRPC/HTTP)
┌──────────────────────────▼──────────────────────────────────────┐
│               Collection / Aggregation Layer                     │
│   ┌─────────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│   │   Prometheus    │  │   Loki       │  │  OTel Collector / │  │
│   │  (pull metrics) │  │  (log agg.)  │  │  Jaeger / Zipkin  │  │
│   └────────┬────────┘  └──────┬───────┘  └─────────┬─────────┘  │
└────────────│──────────────────│──────────────────────│───────────┘
             │                  │                       │
┌────────────▼──────────────────▼───────────────────────▼──────────┐
│                       Storage Layer                               │
│   ┌──────────────┐  ┌────────────────────┐  ┌────────────────┐   │
│   │  Prometheus  │  │  Elasticsearch /   │  │  Object Store  │   │
│   │    TSDB      │  │  Loki chunks       │  │  (S3/GCS for   │   │
│   │  (or Mimir / │  │                    │  │   Tempo/Thanos)│   │
│   │   Thanos)    │  └────────────────────┘  └────────────────┘   │
│   └──────────────┘                                                │
└─────────────────────────────────────────────────────────────────┘
             │                  │                       │
┌────────────▼──────────────────▼───────────────────────▼──────────┐
│                   Visualisation Layer                             │
│   ┌──────────────────────────────────────────────────────────┐   │
│   │           Grafana (dashboards, explore, alerts)          │   │
│   │   Prometheus UI │ Kibana │ Jaeger UI │ Grafana Tempo UI  │   │
│   └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
             │
┌────────────▼──────────────────────────────────────────────────────┐
│                     Alerting Layer                                 │
│   ┌─────────────────┐  ┌──────────────┐  ┌────────────────────┐   │
│   │  Alertmanager   │  │  PagerDuty   │  │    OpsGenie /      │   │
│   │  (routing,      │──▶  (on-call    │  │    VictorOps /     │   │
│   │   grouping,     │  │   paging)    │  │    Slack           │   │
│   │   silencing)    │  └──────────────┘  └────────────────────┘   │
│   └─────────────────┘                                              │
└───────────────────────────────────────────────────────────────────┘
```

## 🔑 Key Concepts

```
Three Pillars of Observability
─────────────────────────────
Metrics  → Aggregated numeric measurements over time (Prometheus, PromQL)
Logs     → Discrete structured event records (Loki, Elasticsearch, structlog)
Traces   → Causal request flow across service boundaries (Jaeger, Tempo, OTel)

Four Golden Signals (Google SRE)
────────────────────────────────
Latency     → How long it takes to serve a request (use p99, not avg)
Traffic     → How much demand is on the system (requests per second)
Errors      → Rate of failed requests (5xx errors / total requests)
Saturation  → How "full" the service is (CPU, memory, queue depth)

RED Method (for services)        USE Method (for infrastructure)
─────────────────────────        ───────────────────────────────
Rate     → requests/second       Utilisation → % of time resource is busy
Errors   → errors/second         Saturation  → work queued / backpressure
Duration → latency distribution  Errors      → error event count

SLI / SLO / SLA
────────────────
SLI  → Service Level Indicator: the actual measurement (e.g., error rate)
SLO  → Service Level Objective: the target (e.g., error rate < 0.1%)
SLA  → Service Level Agreement: the contractual commitment with consequences

OpenTelemetry
─────────────
API      → Vendor-neutral interface for instrumentation (no vendor dependency)
SDK      → Implementation of the API with exporters and processors
Collector → Agent/gateway for receiving, processing, and routing telemetry
```

## 📋 Topics Covered

- **Foundations** — Three pillars, observability vs monitoring, CNCF landscape overview
- **Metrics** — Counter, Gauge, Histogram, Summary; PromQL; recording rules; federation
- **Logging** — Structured logging, log levels, Loki, ELK stack, retention, cardinality
- **Distributed Tracing** — Spans, context propagation, W3C traceparent, Jaeger, Zipkin
- **OpenTelemetry** — API/SDK/Collector, auto-instrumentation, multi-language, OTel Demo
- **Alerting** — Prometheus rules, Alertmanager routing, PagerDuty, symptom-based alerting
- **APM** — Datadog, New Relic, Dynatrace, Grafana Cloud; build vs buy analysis
- **SLIs/SLOs/SLAs** — Error budgets, burn-rate alerting, multi-window alerting
- **Best Practices** — Golden Signals, RED/USE, cardinality management, health checks, dashboard design, observability as code
- **Anti-Patterns** — 14 anti-patterns with code examples, fixes, and maturity mapping

## 🤝 Contributing

This is a living collection of learning resources. Contributions are welcome — see the repository [CONTRIBUTING.md](../CONTRIBUTING.md) for guidelines.

## 🏁 Next Steps

**New to observability?** → Start with [00-OVERVIEW.md](00-OVERVIEW.md) then follow [LEARNING-PATH.md](LEARNING-PATH.md)

**Already know metrics?** → Jump to [03-DISTRIBUTED-TRACING.md](03-DISTRIBUTED-TRACING.md) or [04-OPENTELEMETRY.md](04-OPENTELEMETRY.md)

**Going to production?** → Review [08-BEST-PRACTICES.md](08-BEST-PRACTICES.md) and [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md)

**Want a structured path?** → Follow the [LEARNING-PATH.md](LEARNING-PATH.md) — 6 phases, 10–12 weeks
