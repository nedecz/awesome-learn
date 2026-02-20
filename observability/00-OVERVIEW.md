# Observability Overview

## Table of Contents

1. [Overview](#overview)
2. [What is Observability vs. Monitoring](#what-is-observability-vs-monitoring)
3. [The Three Pillars](#the-three-pillars)
   - [Metrics](#metrics)
   - [Logs](#logs)
   - [Traces](#traces)
4. [Why Observability Matters](#why-observability-matters)
5. [The Four Golden Signals](#the-four-golden-signals)
6. [CNCF Observability Landscape](#cncf-observability-landscape)
7. [Key Tools Overview](#key-tools-overview)
8. [Observability Maturity Model](#observability-maturity-model)
9. [Prerequisites](#prerequisites)
10. [Next Steps](#next-steps)
11. [Version History](#version-history)

---

## Overview

This documentation provides a comprehensive introduction to observability in modern distributed systems. It covers the foundational concepts, tooling landscape, and practical patterns needed to understand and improve system behavior in production.

### Target Audience

- **Developers** building and instrumenting services that need visibility into runtime behavior
- **Site Reliability Engineers (SREs)** responsible for uptime, incident response, and reliability targets
- **DevOps Engineers** building and operating CI/CD pipelines, infrastructure, and platform tooling
- **Architects** designing distributed systems and selecting observability platforms

### Scope

- The three pillars of observability: metrics, logs, and traces
- The four golden signals and how to use them
- Tooling landscape including open-source and commercial options
- The CNCF observability ecosystem
- An observability maturity model for organizations at any stage

---

## What is Observability vs. Monitoring

The terms *monitoring* and *observability* are often used interchangeably, but they represent fundamentally different philosophies about how you understand your systems.

**Monitoring** is the practice of collecting and alerting on predefined metrics and thresholds. You instrument what you know to watch, and you get alerted when something crosses a threshold you've anticipated.

**Observability** is the property of a system that allows you to understand its internal state by examining its external outputs — without having to ship new code or add new instrumentation. An observable system lets you ask *arbitrary questions* about its behavior.

### Comparison Table

| Dimension | Monitoring | Observability |
|---|---|---|
| **Question answered** | "Is the system working?" | "Why is the system not working?" |
| **Data type** | Predefined metrics and thresholds | Metrics, logs, traces, and events (correlated) |
| **Failure mode** | Only catches known failure modes | Helps diagnose unknown failure modes |
| **Interaction style** | Passive alerting, dashboards | Active investigation and exploration |
| **Proactive vs. Reactive** | Primarily reactive (alerts fire after threshold breached) | Enables proactive investigation before users complain |
| **Instrumentation approach** | Instrument what you know to check | Instrument richly so you can ask any question later |
| **Cardinality** | Low-cardinality data to keep costs manageable | High-cardinality data to enable fine-grained queries |
| **Examples** | Nagios, Zabbix, simple Prometheus alerting | OpenTelemetry + Jaeger + Loki + Prometheus (correlated) |

### The Spectrum

```
                    ◄──────────────────────────────────────────────────────►
                    Less Mature                               More Mature

  MONITORING                        OBSERVABILITY
  ┌─────────────────────┐           ┌────────────────────────────────────────┐
  │                     │           │                                        │
  │  ● CPU > 90%? Alert │           │  ● Why is this specific user's         │
  │  ● 500s > 1%? Alert │           │    checkout slow for the last 2 mins?  │
  │  ● Disk full? Alert │           │                                        │
  │                     │           │  ● Trace ID → spans across 5 services  │
  │  Known unknowns     │           │  → logs at each hop → DB slow query    │
  │                     │           │                                        │
  └─────────────────────┘           │  Unknown unknowns                      │
                                    └────────────────────────────────────────┘

  You define the questions             You answer questions you haven't thought of yet
```

---

## The Three Pillars

The three pillars of observability — **Metrics**, **Logs**, and **Traces** — complement each other. Each pillar answers different questions, and together they give you full insight into your system.

```
                         ┌──────────────────────────────────────┐
                         │           Running Service            │
                         │  ┌──────────────────────────────┐   │
                         │  │  request → handler → db →    │   │
                         │  │  cache → response             │   │
                         │  └──────────────────────────────┘   │
                         └──────────┬──────────────────────────┘
                                    │  emits
              ┌─────────────────────┼──────────────────────┐
              │                     │                       │
              ▼                     ▼                       ▼
   ┌──────────────────┐  ┌─────────────────────┐  ┌──────────────────┐
   │     METRICS      │  │       LOGS          │  │     TRACES       │
   │                  │  │                     │  │                  │
   │ Numeric time     │  │ Timestamped         │  │ Causal chain of  │
   │ series data      │  │ structured events   │  │ operations with  │
   │                  │  │                     │  │ timing and       │
   │ "How much?"      │  │ "What happened?"    │  │ context          │
   │ "How fast?"      │  │ "What went wrong?"  │  │                  │
   │ "How often?"     │  │                     │  │ "Where did       │
   │                  │  │                     │  │  time go?"       │
   │  Prometheus      │  │  Loki / ELK         │  │  Jaeger/Zipkin   │
   │  StatsD          │  │  Fluentd            │  │  Tempo           │
   └──────────────────┘  └─────────────────────┘  └──────────────────┘
              │                     │                       │
              └─────────────────────┴───────────────────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │    Grafana / Kibana   │
                         │    Unified Platform   │
                         │    (correlation)      │
                         └──────────────────────┘
```

### Metrics

Metrics are **numeric measurements collected over time**. They are the most efficient form of telemetry to store and query. A metric data point consists of:

- A **metric name** (e.g., `http_requests_total`)
- A **timestamp** (when the measurement was taken)
- A **value** (a float64 number)
- A set of **labels/tags** (key-value pairs providing dimensions, e.g., `method="GET"`, `status="200"`)

**When to use metrics:**
- Dashboards showing aggregate system health
- Alerting on SLOs and SLIs
- Capacity planning and trend analysis
- The four golden signals

**Common tools:** Prometheus, StatsD, InfluxDB, Datadog, CloudWatch, OpenTelemetry Metrics

```
Example Prometheus metric:
http_requests_total{method="GET", path="/api/orders", status="200"} 14298 1687200000000
                    └── labels ─────────────────────────────────┘  └─value─┘ └─timestamp─┘
```

### Logs

Logs are **immutable, time-stamped records of discrete events** that occurred in a system. Unlike metrics (aggregates), logs capture individual events at full fidelity.

**Structured logs** (JSON) are far more useful than unstructured text because they are machine-parseable and easily indexed.

**When to use logs:**
- Debugging individual request failures
- Audit trails and compliance
- Understanding the sequence of events leading to an error
- Correlating with a trace ID to drill into a specific request

**Common tools:** Loki, Elasticsearch/OpenSearch, Splunk, Fluentd, Fluent Bit, Logstash

```json
{
  "timestamp": "2025-01-15T14:32:01.123Z",
  "level": "ERROR",
  "service": "order-service",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "message": "Failed to process payment",
  "user_id": "usr_abc123",
  "order_id": "ord_xyz789",
  "error": "connection timeout after 5000ms"
}
```

### Traces

Distributed traces capture the **end-to-end journey of a request** as it flows through multiple services. A trace is a directed acyclic graph of **spans**, where each span represents a single operation.

**When to use traces:**
- Understanding latency across service boundaries
- Finding which service in a chain is the bottleneck
- Debugging intermittent failures in microservices
- Visualizing service dependencies

**Common tools:** Jaeger, Zipkin, Grafana Tempo, AWS X-Ray, Datadog APM, OpenTelemetry

```
Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736

 0ms         50ms        100ms       150ms       200ms
  │           │           │           │           │
  ├───────────────────────────────────────────────┤  api-gateway       (195ms)
  │   ├───────────────────────────────────────┤      order-service     (160ms)
  │   │   ├─────────────────────────────┤            payment-service   (120ms)
  │   │   │   ├──────────────────┤                   db-query          (80ms)
  │   │   │   └──────────────────┘
  │   │   └─────────────────────────────┘
  │   └───────────────────────────────────────┘
  └───────────────────────────────────────────────┘
```

---

## Why Observability Matters

### Business Value

| Business Outcome | How Observability Contributes |
|---|---|
| **Reduced MTTR** | Engineers can find root causes in minutes instead of hours by correlating traces, logs, and metrics |
| **Improved reliability / uptime** | Proactive detection of degradation before users are impacted at scale |
| **Faster feature delivery** | Confidence to deploy frequently because you can detect and roll back regressions immediately |
| **Cost optimization** | Identify over-provisioned resources, slow queries, and wasteful retry storms |
| **Better on-call experience** | Alerts with rich context reduce alert fatigue and midnight debugging sessions |
| **SLO compliance** | Data-driven error budgets give teams clear signals about reliability posture |
| **Customer trust** | Faster incident detection and resolution reduces user-facing impact |

### Engineering Value

- **Shorter feedback loops**: Know within seconds whether a deployment introduced a regression
- **Reduced cognitive load**: Dashboards and traces surface facts, so engineers spend less time guessing
- **Cross-team collaboration**: Shared observability data lets frontend, backend, SRE, and database teams debug together
- **Post-incident learning**: Rich telemetry enables thorough post-mortems with an accurate timeline
- **Capacity planning**: Trend data drives infrastructure decisions based on evidence, not intuition
- **Service-level accountability**: Teams own their SLOs and can see exactly how their services perform

---

## The Four Golden Signals

The four golden signals — introduced in the [Google SRE Book](https://sre.google/sre-book/monitoring-distributed-systems/) — are the minimum set of metrics every service should expose. If you can only instrument four things, instrument these.

### 1. Latency

> The time it takes to service a request.

It is critical to distinguish between latency of **successful** requests and latency of **failed** requests. A fast error is not a well-performing service.

```promql
# p99 latency for successful HTTP requests over the last 5 minutes
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{status!~"5.."}[5m])) by (le, service)
)
```

| Signal | Example Alert |
|---|---|
| p99 > 500ms | SLO breach — page on-call |
| p50 > 100ms | Trend degradation — ticket for next sprint |

### 2. Traffic

> The amount of demand being placed on your system.

Traffic measures how much your system is being used. Common expressions include requests per second (RPS) for HTTP services, or messages per second for queues.

```promql
# Requests per second across all HTTP endpoints
sum(rate(http_requests_total[1m])) by (service)
```

### 3. Errors

> The rate of requests that fail.

This includes explicit failures (HTTP 5xx), implicit failures (HTTP 200 with wrong content), and policy-based failures (requests taking longer than your SLO threshold).

```promql
# Error rate as a percentage of total traffic
100 * sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
  /
sum(rate(http_requests_total[5m])) by (service)
```

### 4. Saturation

> How "full" your service is. A measure of the fraction of the system that is being consumed.

Saturation predicts upcoming problems. When a resource is near 100% utilization, latency increases and requests start queuing or failing. Common saturation metrics: CPU, memory, disk I/O, thread pool utilization, queue depth.

```promql
# CPU saturation: average CPU usage across all instances
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)

# Memory saturation
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Thread pool saturation
thread_pool_active_threads / thread_pool_max_threads
```

### Golden Signals at a Glance

```
  LATENCY                 TRAFFIC                 ERRORS                  SATURATION
  ┌─────────────┐         ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
  │             │         │  ▲          │         │             │         │ ████████░░  │
  │     p99 ─── │──► SLO  │  │ RPS      │         │  5xx rate   │         │   CPU: 82%  │
  │             │         │  │   ▲      │         │    ▲        │         │             │
  │     p95 ─── │         │  │   │ ▲    │         │    │        │         │ ██████░░░░  │
  │             │         │  └───┴─┴──  │         │    └──────  │         │  Mem: 61%   │
  │     p50 ─── │         │             │         │             │         │             │
  └─────────────┘         └─────────────┘         └─────────────┘         └─────────────┘
   "How slow?"             "How busy?"              "How broken?"           "How full?"
```

---

## CNCF Observability Landscape

The Cloud Native Computing Foundation (CNCF) maintains a broad landscape of observability projects at different stages of maturity.

| Project | Category | CNCF Status | Description |
|---|---|---|---|
| **Prometheus** | Metrics | Graduated | Pull-based metrics collection and alerting with a powerful query language (PromQL) |
| **Jaeger** | Distributed Tracing | Graduated | End-to-end distributed tracing system originally built by Uber |
| **OpenTelemetry** | Instrumentation | Graduated | Vendor-neutral SDK and protocol for collecting metrics, logs, and traces |
| **Fluentd** | Logging | Graduated | Unified logging layer that routes logs from multiple inputs to multiple outputs |
| **Cortex** | Metrics (Long-term storage) | Graduated | Horizontally scalable, multi-tenant Prometheus-as-a-Service |
| **Thanos** | Metrics (Long-term storage) | Graduated | Highly available Prometheus setup with long-term storage capabilities |
| **Grafana** | Visualization | Not in CNCF (CNCF uses it) | De-facto standard for observability dashboards; supports dozens of data sources |
| **Grafana Loki** | Logging | Not in CNCF | Log aggregation system optimized for Kubernetes workloads; uses label-based indexing |
| **Grafana Tempo** | Distributed Tracing | Not in CNCF | Cost-effective, scalable distributed tracing backend |
| **Zipkin** | Distributed Tracing | Not in CNCF | Distributed tracing system open-sourced by Twitter |
| **OpenMetrics** | Metrics Standard | Sandbox | Specification that extends the Prometheus exposition format into a standard |
| **Pixie** | Observability Platform | Sandbox | In-cluster observability for Kubernetes using eBPF |
| **Cilium** | Networking / Observability | Graduated | eBPF-based networking, security, and observability for Kubernetes |
| **Falco** | Security / Observability | Incubating | Cloud-native runtime security and threat detection using eBPF |
| **Vector** | Logs / Metrics | Not in CNCF | High-performance observability data pipeline (Datadog-maintained) |

---

## Key Tools Overview

| Tool | Type | License | Use Case |
|---|---|---|---|
| **Prometheus** | Metrics collection + TSDB | Apache 2.0 | Pull-based metrics scraping, PromQL querying, alerting rules |
| **Grafana** | Visualization | AGPL 3.0 (OSS) | Unified dashboards across Prometheus, Loki, Tempo, and 40+ data sources |
| **Jaeger** | Distributed tracing | Apache 2.0 | Microservice trace collection, storage, and visualization |
| **Zipkin** | Distributed tracing | Apache 2.0 | Lightweight tracing; compatible with many client libraries |
| **OpenTelemetry** | Instrumentation framework | Apache 2.0 | Vendor-neutral SDKs and Collector for all three pillars |
| **Grafana Loki** | Log aggregation | AGPL 3.0 (OSS) | Kubernetes-native log storage with LogQL; low cost due to no full-text indexing |
| **Elasticsearch** | Search + log storage | Elastic License 2.0 | Full-text log search and analytics; core of the ELK/EFK stack |
| **Logstash** | Log pipeline | Elastic License 2.0 | Data ingestion, transformation, and enrichment for Elasticsearch |
| **Kibana** | Log visualization | Elastic License 2.0 | Dashboards, log search UI, and alerting for Elasticsearch data |
| **Fluentd** | Log shipper | Apache 2.0 | Flexible log collection and routing; CNCF Graduated project |
| **Fluent Bit** | Lightweight log shipper | Apache 2.0 | Low-resource-footprint log forwarder; ideal for Kubernetes DaemonSets |
| **Datadog** | Commercial APM platform | Commercial | All-in-one metrics, logs, traces, RUM, and synthetic monitoring |
| **New Relic** | Commercial APM platform | Commercial | Full-stack observability with generous free tier; strong APM and NRQL |
| **Dynatrace** | Commercial APM platform | Commercial | AI-driven observability with automatic topology discovery and root cause analysis |

---

## Observability Maturity Model

Organizations adopt observability incrementally. The maturity model below describes five levels, from no observability to full, proactive observability practice.

| Level | Name | Characteristics | Typical State |
|---|---|---|---|
| **0** | Dark | No metrics, no logs, no tracing. Failures discovered by users or manual checks. | Legacy monoliths, early-stage startups, embedded systems |
| **1** | Reactive | Basic server metrics (CPU, memory, disk). Unstructured application logs. Alerts fire after users are already impacted. | Teams relying on `top`, `tail -f`, and manual restarts |
| **2** | Structured | Structured logging, basic Prometheus metrics for golden signals, Grafana dashboards. Alerts are tuned but still mostly reactive. | Teams using a shared monitoring stack, alert runbooks exist |
| **3** | Correlated | All three pillars instrumented. Trace IDs in logs. Dashboards link metrics → logs → traces. SLOs defined and tracked. Error budgets active. | Platform teams with dedicated observability infrastructure |
| **4** | Proactive | Anomaly detection, ML-based alerting, continuous profiling, automated runbooks, chaos engineering. Observability drives architecture decisions. | Mature SRE organizations, hyperscalers, observability-as-product teams |

```
  Maturity ▲
           │                                            ┌─────────────┐
     4     │                                            │  Proactive  │
           │                                      ┌────┴────────┐    │
     3     │                                      │  Correlated │    │
           │                              ┌───────┴─────────┐   │    │
     2     │                              │   Structured    │   │    │
           │                    ┌─────────┴──────────┐      │   │    │
     1     │                    │    Reactive        │      │   │    │
           │         ┌──────────┴──────────┐         │      │   │    │
     0     │         │       Dark          │         │      │   │    │
           └─────────┴────────────────────────────────────────────────►
                                                                    Time / Effort
```

---

## Prerequisites

### Required Knowledge

Before working through the observability series, you should be familiar with:

| Topic | Why It Matters |
|---|---|
| **Linux command line** | Navigating logs, running agents, inspecting processes |
| **Docker and containers** | Running observability tools locally, understanding container logs |
| **Kubernetes basics** | Most examples use Kubernetes; understanding Pods, Services, and namespaces helps |
| **HTTP and REST APIs** | Prometheus scrapes HTTP endpoints; tracing propagates via HTTP headers |
| **Basic networking** | Port forwarding, DNS, load balancers — observability agents communicate over the network |
| **A backend programming language** | Go or Python preferred for instrumentation examples |

### Required Tools

Install the following tools to follow hands-on examples:

```bash
# Docker Desktop or Docker Engine
# macOS
brew install --cask docker

# Ubuntu/Debian
sudo apt-get update && sudo apt-get install docker-ce docker-ce-cli containerd.io

# Docker Compose (included with Docker Desktop; standalone install for Linux)
sudo apt-get install docker-compose-plugin
# or
pip install docker-compose

# kubectl
# macOS
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Helm (for Kubernetes chart deployments)
# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# k3d or kind (local Kubernetes cluster)
# k3d
brew install k3d       # macOS
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash   # Linux

# kind
brew install kind      # macOS
go install sigs.k8s.io/kind@latest   # Go required

# grpcurl (for inspecting gRPC services)
brew install grpcurl   # macOS
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest  # Go

# jq (for parsing JSON log output)
brew install jq        # macOS
sudo apt-get install jq  # Ubuntu
```

### Recommended: Local Observability Stack

Use the following `docker-compose.yml` as a starting point for all exercises:

```yaml
version: "3.8"
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # Jaeger UI
      - "4317:4317"    # OTLP gRPC
      - "4318:4318"    # OTLP HTTP

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/log:/var/log
      - ./promtail-config.yaml:/etc/promtail/config.yml
```

```bash
# Start the local stack
docker compose up -d

# Verify all services are running
docker compose ps

# Access UIs
# Prometheus: http://localhost:9090
# Grafana:    http://localhost:3000  (admin / admin)
# Jaeger UI:  http://localhost:16686
```

---

## Next Steps

Work through the observability series in order, or jump to the topic most relevant to you:

| File | Topic | Description |
|---|---|---|
| [01-METRICS.md](01-METRICS.md) | Metrics | Prometheus metric types, PromQL, instrumentation, Grafana dashboards |
| [02-LOGGING.md](02-LOGGING.md) | Logging | Structured logging, ELK stack, Grafana Loki, Fluentd, Fluent Bit |
| [03-DISTRIBUTED-TRACING.md](03-DISTRIBUTED-TRACING.md) | Distributed Tracing | Jaeger, Zipkin, OpenTelemetry, context propagation, sampling |
| [04-OPENTELEMETRY.md](04-OPENTELEMETRY.md) | OpenTelemetry | OTel SDK, Collector, auto-instrumentation, exporters |
| [05-ALERTING.md](05-ALERTING.md) | Alerting | Alertmanager, alert rules, routing, silences, PagerDuty/Slack integration |
| [06-SLOS-AND-SLIS.md](06-SLOS-AND-SLIS.md) | SLOs and SLIs | Service Level Objectives, error budgets, burn rates, burn rate alerts |
| [07-DASHBOARDS.md](07-DASHBOARDS.md) | Dashboards | Grafana dashboard design, variables, annotations, best practices |
| [08-INCIDENT-RESPONSE.md](08-INCIDENT-RESPONSE.md) | Incident Response | Runbooks, war rooms, post-mortems, blameless culture |
| [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md) | Anti-Patterns | Common observability mistakes and how to avoid them |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Structured learning path by role and experience level |

### Suggested Learning Path by Role

```
Developer:
  00-OVERVIEW → 01-METRICS → 02-LOGGING → 03-DISTRIBUTED-TRACING → 04-OPENTELEMETRY

SRE / Platform Engineer:
  00-OVERVIEW → 01-METRICS → 05-ALERTING → 06-SLOS-AND-SLIS → 07-DASHBOARDS → 08-INCIDENT-RESPONSE

Architect:
  00-OVERVIEW → 04-OPENTELEMETRY → 06-SLOS-AND-SLIS → 09-ANTI-PATTERNS → LEARNING-PATH
```

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial observability overview documentation |
