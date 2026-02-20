# OpenTelemetry

## Table of Contents

1. [Overview](#overview)
2. [OTel Architecture](#otel-architecture)
3. [OTel Components](#otel-components)
   - [API](#api)
   - [SDK](#sdk)
   - [Collector](#collector)
   - [Exporters](#exporters-component)
   - [Semantic Conventions](#semantic-conventions-component)
4. [Auto-Instrumentation](#auto-instrumentation)
   - [Java Agent](#java-agent)
   - [Python](#python-auto-instrumentation)
   - [Node.js](#nodejs-auto-instrumentation)
5. [Manual Instrumentation](#manual-instrumentation)
   - [Go](#go-manual-instrumentation)
   - [Python](#python-manual-instrumentation)
   - [Span Links](#span-links)
6. [OTLP Collector Configuration](#otlp-collector-configuration)
7. [Exporters Configuration](#exporters-configuration)
8. [Semantic Conventions](#semantic-conventions)
9. [Propagators](#propagators)
10. [OTel Demo Architecture](#otel-demo-architecture)
11. [Migration Guide](#migration-guide)
    - [From OpenTracing](#from-opentracing)
    - [From OpenCensus](#from-opencensus)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

OpenTelemetry (OTel) is a **vendor-neutral, open-source observability framework** for generating, collecting, and exporting telemetry data — traces, metrics, and logs — from cloud-native applications. It is the result of merging two earlier CNCF projects:

- **OpenTracing** (2016): A vendor-neutral API for distributed tracing. Defined interfaces but not implementations. Supported Jaeger, Zipkin, Lightstep, and others.
- **OpenCensus** (2018): Google's framework for collecting traces and metrics. Included both API and SDK, but was tightly coupled to Google Cloud.

### Why OpenTelemetry Was Created

| Problem | OpenTracing Response | OpenCensus Response | OTel Solution |
|---------|---------------------|---------------------|---------------|
| Vendor lock-in | Neutral tracing API | Neutral API+SDK | Unified API+SDK+Collector |
| Traces only | Traces only | Traces + Metrics | Traces + Metrics + Logs |
| No SDK standard | API only | SDK included | Standardized SDK spec |
| Fragmented ecosystems | Separate project | Separate project | Single CNCF project |
| No data collection agent | None | None | OTel Collector |

In **2019**, the OpenTracing and OpenCensus maintainers announced they would merge into OpenTelemetry under the CNCF umbrella. OpenTelemetry reached **GA (General Availability)** for traces and metrics in 2021, with logs reaching GA in 2023.

### CNCF Project Status

OpenTelemetry is a **CNCF Graduated project** (graduated in 2021), making it one of the most important observability standards in the cloud-native ecosystem. It is the second-most active CNCF project after Kubernetes by contributor count.

```
CNCF Graduation Timeline:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

2019-05  OpenTelemetry announced (OpenTracing + OpenCensus merge)
2019-09  CNCF Sandbox
2021-08  Traces + Metrics GA in major languages
2021-11  CNCF Graduated
2023-02  Logs specification GA
2024-Q1  Profiles specification (experimental)
```

---

## OTel Architecture

The OpenTelemetry architecture separates concerns cleanly: your application code uses the **API**, the **SDK** implements the API and handles batching/sampling, the **Collector** receives, processes, and routes telemetry, and **backends** store and visualize data.

```
OTel End-to-End Architecture
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────────────────────────────────────────────────────────────┐
  │                      YOUR APPLICATION                               │
  │                                                                     │
  │   ┌──────────────────────┐    ┌───────────────────────────────────┐ │
  │   │    Business Logic    │    │   Auto-Instrumentation (agent)    │ │
  │   │  tracer.start_span() │    │   HTTP, DB, gRPC, Redis, etc.    │ │
  │   └──────────┬───────────┘    └────────────────┬──────────────────┘ │
  │              │                                 │                     │
  │              ▼                                 ▼                     │
  │   ┌──────────────────────────────────────────────────────────────┐  │
  │   │                    OTel API (facade)                         │  │
  │   │   TracerProvider  |  MeterProvider  |  LoggerProvider        │  │
  │   └──────────────────────────┬───────────────────────────────────┘  │
  │                              │                                       │
  │                              ▼                                       │
  │   ┌──────────────────────────────────────────────────────────────┐  │
  │   │                    OTel SDK (implementation)                  │  │
  │   │   Sampler  |  SpanProcessor  |  MetricReader  |  LogProcessor │  │
  │   │   Exporter configuration  |  Resource detection              │  │
  │   └──────────────────────────┬───────────────────────────────────┘  │
  └──────────────────────────────┼──────────────────────────────────────┘
                                 │
                    OTLP (gRPC or HTTP/protobuf)
                                 │
                                 ▼
  ┌──────────────────────────────────────────────────────────────────────┐
  │                     OTel Collector                                    │
  │                                                                       │
  │  Receivers         Processors           Exporters                    │
  │  ┌──────────┐   ┌──────────────────┐   ┌──────────────────────────┐ │
  │  │   OTLP   │   │  batch           │   │  Jaeger       (traces)   │ │
  │  │ Prometheus│──▶│  memory_limiter  │──▶│  Prometheus   (metrics)  │ │
  │  │  Jaeger  │   │  resourcedetect  │   │  Zipkin       (traces)   │ │
  │  │  Zipkin  │   │  attributes      │   │  Datadog      (all)      │ │
  │  │  Kafka   │   │  tail_sampling   │   │  OTLP/gRPC    (all)      │ │
  │  └──────────┘   └──────────────────┘   │  Logging      (debug)    │ │
  │                                        └──────────────────────────┘ │
  └──────────────────────────────────────────────────────────────────────┘
              │              │              │            │
              ▼              ▼              ▼            ▼
          ┌──────┐      ┌──────────┐   ┌───────┐   ┌─────────┐
          │Jaeger│      │Prometheus│   │Zipkin │   │Datadog  │
          │  UI  │      │+ Grafana │   │  UI   │   │ Backend │
          └──────┘      └──────────┘   └───────┘   └─────────┘
```

---

## OTel Components

### API

The **OTel API** provides language-agnostic interfaces for instrumentation. It is intentionally minimal and ships with **no-op (no-operation) implementations** by default — meaning if no SDK is configured, API calls silently do nothing without errors or performance overhead.

**Why this matters:** Libraries and frameworks can safely add OTel API calls without forcing their users to adopt OTel. The API calls become active only when a user configures an SDK.

```
API No-Op Behavior
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Library code (uses API):          SDK not installed:
  ┌─────────────────────────┐       ┌──────────────────────────┐
  │ tracer = get_tracer()   │       │ Returns NoOpTracer        │
  │ span = tracer.start()   │──────▶│ span.set_attribute() = ∅  │
  │ span.set_attribute(...) │       │ span.end() = ∅            │
  │ span.end()              │       │ Zero overhead             │
  └─────────────────────────┘       └──────────────────────────┘

  SDK installed:
  ┌──────────────────────────┐
  │ Returns real SDK tracer   │
  │ span is actually recorded │
  │ Exported to backend       │
  └──────────────────────────┘
```

**API Components by Signal:**

| Signal | API Interface | Key Operations |
|--------|--------------|----------------|
| Traces | `TracerProvider`, `Tracer`, `Span` | `start_span`, `set_attribute`, `add_event`, `set_status`, `end` |
| Metrics | `MeterProvider`, `Meter`, `Instrument` | `create_counter`, `create_histogram`, `create_gauge`, `record` |
| Logs | `LoggerProvider`, `Logger`, `LogRecord` | `emit`, structured key-value body |
| Baggage | `Baggage`, `BaggageEntry` | `set_value`, `get_value`, propagate across services |
| Context | `Context`, `Propagator` | `attach`, `detach`, `extract`, `inject` |

### SDK

The **OTel SDK** is the concrete implementation of the API. It handles:

- **Sampling**: deciding which traces to record (always-on, probability, tail-based)
- **SpanProcessors**: synchronous or asynchronous processing before export
- **Exporters**: sending data to backends in OTLP, Jaeger, Zipkin, or other formats
- **Resource detection**: auto-detecting service name, host, container, cloud provider
- **Configuration**: via code, environment variables, or config files

```
SDK Internal Pipeline (Traces)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 Tracer.start_span()
        │
        ▼
 ┌──────────────┐
 │   Sampler    │ ← Should this span be recorded?
 │ (head-based) │   AlwaysOn / AlwaysOff / TraceIdRatioBased
 └──────┬───────┘
        │ YES (sampled)
        ▼
 ┌──────────────────────────────────────┐
 │         Span (in memory)             │
 │  traceId, spanId, parentSpanId       │
 │  name, kind, startTime, endTime      │
 │  attributes[], events[], links[]     │
 │  status, resource                    │
 └──────────────────┬───────────────────┘
                    │ span.end()
                    ▼
 ┌──────────────────────────────────────┐
 │         SpanProcessor                │
 │                                      │
 │  SimpleSpanProcessor → export sync   │
 │  BatchSpanProcessor  → buffer+export │
 └──────────────────┬───────────────────┘
                    │
                    ▼
 ┌──────────────────────────────────────┐
 │            Exporter                  │
 │  OTLP / Jaeger / Zipkin / Console    │
 └──────────────────────────────────────┘
```

**SDK Configuration via Environment Variables:**

```bash
# Core configuration
export OTEL_SERVICE_NAME="payment-service"
export OTEL_SERVICE_VERSION="1.2.3"
export OTEL_RESOURCE_ATTRIBUTES="deployment.environment=production,team=payments"

# Exporter
export OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4317"
export OTEL_EXPORTER_OTLP_PROTOCOL="grpc"          # grpc | http/protobuf | http/json

# Traces
export OTEL_TRACES_EXPORTER="otlp"
export OTEL_TRACES_SAMPLER="parentbased_traceidratio"
export OTEL_TRACES_SAMPLER_ARG="0.1"               # 10% sampling

# Metrics
export OTEL_METRICS_EXPORTER="otlp"
export OTEL_METRIC_EXPORT_INTERVAL="60000"          # ms

# Logs
export OTEL_LOGS_EXPORTER="otlp"

# Propagators
export OTEL_PROPAGATORS="tracecontext,baggage"

# SDK disabled (for testing)
export OTEL_SDK_DISABLED="false"
```

### Collector

The **OTel Collector** is a standalone proxy/agent that receives telemetry data, processes it, and exports it to one or more backends. It can run as a **sidecar**, **daemonset**, or **standalone gateway**.

**Why use a Collector instead of direct export?**

| Reason | Explanation |
|--------|-------------|
| Vendor decoupling | Change backends without redeploying apps |
| Data enrichment | Add k8s labels, environment tags without touching app code |
| Filtering/sampling | Drop noisy spans at collection time |
| Fan-out | Send same data to multiple backends simultaneously |
| Buffering | Handle backend unavailability gracefully |
| Protocol translation | Receive Jaeger, export as OTLP |
| Cost control | Tail-based sampling reduces trace volume before billing |

```
OTel Collector Internals
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ┌─────────────────────────────────────────────────────────────────┐
  │                     OTel Collector Process                       │
  │                                                                  │
  │  ┌─────────────────┐                                            │
  │  │    RECEIVERS     │  Accept data in various formats:          │
  │  │                  │                                            │
  │  │  otlp (grpc)    │◀── port 4317 (apps, other collectors)     │
  │  │  otlp (http)    │◀── port 4318                               │
  │  │  jaeger         │◀── port 14250/14268                        │
  │  │  zipkin         │◀── port 9411                               │
  │  │  prometheus     │◀── scrape targets                          │
  │  │  kafka          │◀── Kafka topics                            │
  │  │  filelog        │◀── log files on disk                       │
  │  └────────┬────────┘                                            │
  │           │                                                      │
  │           ▼                                                      │
  │  ┌─────────────────┐                                            │
  │  │   PROCESSORS    │  Transform, filter, enrich data:           │
  │  │                 │                                             │
  │  │  batch          │  Buffer spans, reduce API calls            │
  │  │  memory_limiter │  Prevent OOM crashes                       │
  │  │  resourcedetect │  Add host/container/cloud attributes       │
  │  │  attributes     │  Add/remove/rename attributes              │
  │  │  filter         │  Drop unwanted telemetry                   │
  │  │  tail_sampling  │  Sample based on full trace                │
  │  │  spanmetrics    │  Derive RED metrics from traces            │
  │  └────────┬────────┘                                            │
  │           │                                                      │
  │           ▼                                                      │
  │  ┌─────────────────┐                                            │
  │  │    EXPORTERS    │  Send data to backends:                    │
  │  │                 │                                             │
  │  │  otlp           │──▶ Any OTLP-compatible backend             │
  │  │  jaeger         │──▶ Jaeger collector                        │
  │  │  prometheus     │──▶ Prometheus scrape endpoint              │
  │  │  zipkin         │──▶ Zipkin backend                          │
  │  │  datadog        │──▶ Datadog                                 │
  │  │  elasticsearch  │──▶ Elasticsearch / Elastic APM             │
  │  │  logging        │──▶ stdout (debugging)                      │
  │  └─────────────────┘                                            │
  │                                                                  │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  SERVICE PIPELINES (wire receivers → processors → exporters) │
  │  │  traces:  [otlp] → [batch, memory_limiter] → [jaeger, otlp]│ │
  │  │  metrics: [otlp, prometheus] → [batch] → [prometheus]     │  │
  │  │  logs:    [otlp, filelog] → [batch] → [elasticsearch]     │  │
  │  └──────────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────────┘
```

### Exporters Component

Exporters ship telemetry data from the SDK or Collector to backends.

| Exporter | Protocol | Primary Backend | Signal Support |
|----------|----------|----------------|----------------|
| `otlp/grpc` | gRPC + Protobuf | Any OTLP backend | Traces, Metrics, Logs |
| `otlp/http` | HTTP + Protobuf or JSON | Any OTLP backend | Traces, Metrics, Logs |
| `jaeger` | Jaeger Thrift/gRPC | Jaeger | Traces only |
| `zipkin` | Zipkin JSON | Zipkin | Traces only |
| `prometheus` | Prometheus text format | Prometheus | Metrics only |
| `datadog` | Datadog API | Datadog | Traces, Metrics, Logs |
| `newrelic` | New Relic OTLP | New Relic | Traces, Metrics, Logs |
| `elasticsearch` | Elasticsearch API | Elastic Stack | Traces, Metrics, Logs |
| `loki` | Loki push API | Grafana Loki | Logs only |
| `kafka` | Kafka Protocol | Kafka topics | Traces, Metrics, Logs |
| `stdout/logging` | stdout | Terminal | Traces, Metrics, Logs |
| `file` | JSON/protobuf file | Filesystem | Traces, Metrics, Logs |

### Semantic Conventions Component

**Semantic Conventions** are standardized attribute names and values for common operations. They allow different instrumentation libraries to produce consistent, queryable data.

**Why they matter:**

```
WITHOUT Semantic Conventions:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Library A records:  "http_method" = "GET"
  Library B records:  "method"      = "GET"
  Library C records:  "verb"        = "GET"
  Library D records:  "HTTP_METHOD" = "GET"

  → Querying "all GET requests" requires knowing every convention.
  → Dashboard panels break when you switch libraries.

WITH Semantic Conventions:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  All libraries record: "http.request.method" = "GET"
  → One query works everywhere.
  → Dashboards are portable.
  → Alerting rules are consistent.
```

---

## Auto-Instrumentation

Auto-instrumentation injects tracing and metrics code into your application without modifying source code. It works by patching popular libraries (HTTP clients, database drivers, messaging systems) at load time.

### Java Agent

The Java agent uses bytecode manipulation (Java Instrumentation API) to instrument applications at startup.

**Basic usage:**
```bash
# Download the agent
curl -Lo opentelemetry-javaagent.jar \
  https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar

# Run your application with the agent
java -javaagent:/path/to/opentelemetry-javaagent.jar \
     -Dotel.service.name=payment-service \
     -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
     -Dotel.traces.exporter=otlp \
     -Dotel.metrics.exporter=otlp \
     -Dotel.logs.exporter=otlp \
     -jar myapp.jar
```

**Using environment variables (preferred for containers):**
```bash
export OTEL_SERVICE_NAME=payment-service
export OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
export OTEL_TRACES_EXPORTER=otlp
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export JAVA_TOOL_OPTIONS="-javaagent:/opt/opentelemetry-javaagent.jar"

java -jar myapp.jar
```

**Docker:**
```dockerfile
FROM eclipse-temurin:21-jre

# Download OTel Java agent
RUN curl -Lo /opt/opentelemetry-javaagent.jar \
    https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar

COPY target/myapp.jar /app/myapp.jar

ENV JAVA_TOOL_OPTIONS="-javaagent:/opt/opentelemetry-javaagent.jar"
ENV OTEL_SERVICE_NAME="payment-service"
ENV OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4317"
ENV OTEL_TRACES_EXPORTER="otlp"

CMD ["java", "-jar", "/app/myapp.jar"]
```

**Kubernetes with init container:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    spec:
      initContainers:
        - name: otel-agent-init
          image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
          command: ["cp", "/javaagent.jar", "/otel-auto-instrumentation/javaagent.jar"]
          volumeMounts:
            - name: otel-auto-instrumentation
              mountPath: /otel-auto-instrumentation

      containers:
        - name: payment-service
          image: myregistry/payment-service:1.2.3
          env:
            - name: JAVA_TOOL_OPTIONS
              value: "-javaagent:/otel-auto-instrumentation/javaagent.jar"
            - name: OTEL_SERVICE_NAME
              value: "payment-service"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://otel-collector.observability:4317"
            - name: OTEL_TRACES_EXPORTER
              value: "otlp"
            - name: OTEL_METRICS_EXPORTER
              value: "otlp"
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "k8s.namespace.name=$(NAMESPACE),k8s.pod.name=$(POD_NAME)"
          env:
            - name: NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          volumeMounts:
            - name: otel-auto-instrumentation
              mountPath: /otel-auto-instrumentation

      volumes:
        - name: otel-auto-instrumentation
          emptyDir: {}
```

**Using OTel Operator (recommended for Kubernetes):**
```yaml
# First install the operator, then annotate your deployment:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  annotations:
    instrumentation.opentelemetry.io/inject-java: "true"
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-java: "true"
    spec:
      containers:
        - name: payment-service
          image: myregistry/payment-service:1.2.3
```

### Python Auto-Instrumentation

```bash
# Install auto-instrumentation packages
pip install opentelemetry-distro opentelemetry-exporter-otlp

# Bootstrap installs all available instrumentation libraries
opentelemetry-bootstrap -a install

# Run your application with auto-instrumentation
opentelemetry-instrument \
    --traces_exporter otlp \
    --metrics_exporter otlp \
    --logs_exporter otlp \
    --exporter_otlp_endpoint http://otel-collector:4317 \
    --service_name payment-service \
    python app.py
```

**Environment variable approach:**
```bash
export OTEL_SERVICE_NAME=payment-service
export OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_TRACES_EXPORTER=otlp
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_PYTHON_LOG_CORRELATION=true
export OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true

opentelemetry-instrument python app.py
```

**Libraries auto-instrumented by `opentelemetry-bootstrap`:**

| Library | Package |
|---------|---------|
| Django | `opentelemetry-instrumentation-django` |
| Flask | `opentelemetry-instrumentation-flask` |
| FastAPI | `opentelemetry-instrumentation-fastapi` |
| SQLAlchemy | `opentelemetry-instrumentation-sqlalchemy` |
| requests | `opentelemetry-instrumentation-requests` |
| httpx | `opentelemetry-instrumentation-httpx` |
| Redis | `opentelemetry-instrumentation-redis` |
| Celery | `opentelemetry-instrumentation-celery` |
| psycopg2 | `opentelemetry-instrumentation-psycopg2` |
| pymongo | `opentelemetry-instrumentation-pymongo` |

### Node.js Auto-Instrumentation

**Installation:**
```bash
npm install @opentelemetry/sdk-node \
            @opentelemetry/auto-instrumentations-node \
            @opentelemetry/exporter-trace-otlp-grpc \
            @opentelemetry/exporter-metrics-otlp-grpc
```

**`tracing.js` — complete setup file (load before your app):**
```javascript
'use strict';

const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { OTLPMetricExporter } = require('@opentelemetry/exporter-metrics-otlp-grpc');
const { PeriodicExportingMetricReader } = require('@opentelemetry/sdk-metrics');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

const resource = new Resource({
  [SemanticResourceAttributes.SERVICE_NAME]: process.env.OTEL_SERVICE_NAME || 'payment-service',
  [SemanticResourceAttributes.SERVICE_VERSION]: process.env.npm_package_version || '0.0.0',
  'deployment.environment': process.env.NODE_ENV || 'development',
});

const traceExporter = new OTLPTraceExporter({
  url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://localhost:4317',
});

const metricExporter = new OTLPMetricExporter({
  url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://localhost:4317',
});

const sdk = new NodeSDK({
  resource,
  traceExporter,
  metricReader: new PeriodicExportingMetricReader({
    exporter: metricExporter,
    exportIntervalMillis: 60_000,
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      // Configure individual instrumentations
      '@opentelemetry/instrumentation-http': {
        ignoreIncomingRequestHook: (req) => req.url === '/healthz',
      },
      '@opentelemetry/instrumentation-express': { enabled: true },
      '@opentelemetry/instrumentation-pg': { enhancedDatabaseReporting: true },
      '@opentelemetry/instrumentation-redis': { enabled: true },
    }),
  ],
});

sdk.start();

// Graceful shutdown
process.on('SIGTERM', () => {
  sdk.shutdown()
    .then(() => console.log('OTel SDK shut down successfully'))
    .catch((error) => console.error('Error shutting down OTel SDK', error))
    .finally(() => process.exit(0));
});
```

**Using the `--require` flag:**
```bash
# Start with auto-instrumentation
node --require ./tracing.js app.js

# Or in package.json:
{
  "scripts": {
    "start": "node --require ./tracing.js app.js",
    "start:prod": "NODE_ENV=production node --require ./tracing.js app.js"
  }
}
```

**Without a setup file (env vars only):**
```bash
# The register pattern — zero-code setup
node --require @opentelemetry/auto-instrumentations-node/register app.js

export OTEL_SERVICE_NAME=payment-service
export OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
export OTEL_NODE_ENABLED_INSTRUMENTATIONS=http,express,pg,redis
```

---

## Manual Instrumentation

Manual instrumentation gives you fine-grained control over what gets traced and what attributes are attached.

### Go Manual Instrumentation

```bash
# Install OTel Go packages
go get go.opentelemetry.io/otel
go get go.opentelemetry.io/otel/trace
go get go.opentelemetry.io/otel/sdk/trace
go get go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc
go get go.opentelemetry.io/otel/attribute
go get go.opentelemetry.io/otel/codes
```

**Complete Go tracing setup and usage:**
```go
package main

import (
    "context"
    "errors"
    "fmt"
    "log"
    "time"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.26.0"
    "go.opentelemetry.io/otel/trace"
)

// initTracer configures the OTel SDK and returns a shutdown function.
func initTracer(ctx context.Context) (func(context.Context) error, error) {
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("otel-collector:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, fmt.Errorf("creating OTLP exporter: %w", err)
    }

    res, err := resource.New(ctx,
        resource.WithAttributes(
            semconv.ServiceName("payment-service"),
            semconv.ServiceVersion("1.2.3"),
            attribute.String("deployment.environment", "production"),
        ),
        resource.WithProcess(),
        resource.WithOS(),
        resource.WithHost(),
    )
    if err != nil {
        return nil, fmt.Errorf("creating resource: %w", err)
    }

    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.ParentBased(
            sdktrace.TraceIDRatioBased(0.1), // 10% sampling
        )),
    )
    otel.SetTracerProvider(tp)

    return tp.Shutdown, nil
}

// Creating a tracer — use the package/component name as the instrumentation scope
var tracer = otel.Tracer("github.com/myorg/payment-service/payments")

// ProcessPayment demonstrates comprehensive span usage.
func ProcessPayment(ctx context.Context, orderID string, amount float64) error {
    // Start a span — always defer End() immediately after
    ctx, span := tracer.Start(ctx, "ProcessPayment",
        trace.WithSpanKind(trace.SpanKindInternal),
        trace.WithAttributes(
            attribute.String("order.id", orderID),
            attribute.Float64("payment.amount", amount),
            attribute.String("payment.currency", "USD"),
        ),
    )
    defer span.End()

    // Add attributes dynamically
    span.SetAttributes(attribute.String("payment.processor", "stripe"))

    // Record an event (a timestamped annotation)
    span.AddEvent("payment.validation.started")

    if amount <= 0 {
        err := errors.New("invalid payment amount")
        // Record the error as a span event with stack trace
        span.RecordError(err,
            trace.WithAttributes(attribute.String("validation.field", "amount")),
        )
        // Set span status to Error
        span.SetStatus(codes.Error, "validation failed: amount must be positive")
        return err
    }

    span.AddEvent("payment.validation.passed",
        trace.WithAttributes(attribute.Float64("validated.amount", amount)),
    )

    // Create a child span for the fraud check
    if err := checkFraud(ctx, orderID, amount); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "fraud check failed")
        return fmt.Errorf("fraud check: %w", err)
    }

    // Create a child span for the database write
    if err := savePayment(ctx, orderID, amount); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "database error")
        return fmt.Errorf("saving payment: %w", err)
    }

    // Mark span as successful explicitly
    span.SetStatus(codes.Ok, "")
    span.AddEvent("payment.completed",
        trace.WithAttributes(
            attribute.String("order.id", orderID),
            attribute.String("payment.status", "success"),
        ),
    )
    return nil
}

// checkFraud is a child span example.
func checkFraud(ctx context.Context, orderID string, amount float64) error {
    // ctx already carries the parent span context
    ctx, span := tracer.Start(ctx, "FraudCheck",
        trace.WithSpanKind(trace.SpanKindClient),
        trace.WithAttributes(
            attribute.String("peer.service", "fraud-service"),
            attribute.String("order.id", orderID),
        ),
    )
    defer span.End()

    start := time.Now()
    // ... call fraud service ...
    duration := time.Since(start)

    span.SetAttributes(
        attribute.Bool("fraud.detected", false),
        attribute.Float64("fraud.score", 0.12),
        attribute.Int64("fraud.check.duration_ms", duration.Milliseconds()),
    )
    return nil
}

// savePayment shows database span conventions.
func savePayment(ctx context.Context, orderID string, amount float64) error {
    ctx, span := tracer.Start(ctx, "db.payments.insert",
        trace.WithSpanKind(trace.SpanKindClient),
        trace.WithAttributes(
            // Semantic conventions for DB spans
            semconv.DBSystemPostgreSQL,
            semconv.DBName("payments_db"),
            semconv.DBOperation("INSERT"),
            semconv.DBSQLTable("payments"),
            attribute.String("db.statement", "INSERT INTO payments (order_id, amount) VALUES (?, ?)"),
        ),
    )
    defer span.End()

    // ... execute DB query ...
    _ = ctx
    return nil
}

func main() {
    ctx := context.Background()
    shutdown, err := initTracer(ctx)
    if err != nil {
        log.Fatal(err)
    }
    defer func() {
        if err := shutdown(ctx); err != nil {
            log.Printf("error shutting down tracer: %v", err)
        }
    }()

    if err := ProcessPayment(ctx, "order-123", 49.99); err != nil {
        log.Printf("payment error: %v", err)
    }
}
```

### Python Manual Instrumentation

```python
import time
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION
from opentelemetry.trace import Status, StatusCode
from opentelemetry.semconv.trace import SpanAttributes


def init_tracer():
    """Configure the OpenTelemetry SDK."""
    resource = Resource.create({
        SERVICE_NAME: "payment-service",
        SERVICE_VERSION: "1.2.3",
        "deployment.environment": "production",
    })

    exporter = OTLPSpanExporter(
        endpoint="http://otel-collector:4317",
        insecure=True,
    )

    provider = TracerProvider(resource=resource)
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)
    return provider


# Get a tracer for this module
tracer = trace.get_tracer(__name__, "1.2.3")


def process_payment(order_id: str, amount: float) -> dict:
    """Process a payment with full tracing."""
    with tracer.start_as_current_span(
        "ProcessPayment",
        kind=trace.SpanKind.INTERNAL,
        attributes={
            "order.id": order_id,
            "payment.amount": amount,
            "payment.currency": "USD",
        }
    ) as span:
        span.add_event("payment.validation.started")

        if amount <= 0:
            error = ValueError(f"Invalid amount: {amount}")
            span.record_exception(error)
            span.set_status(Status(StatusCode.ERROR, "Validation failed"))
            raise error

        span.add_event("payment.validation.passed")
        span.set_attribute("payment.processor", "stripe")

        # Nested child span for external call
        charge_result = _charge_stripe(order_id, amount)

        # Nested child span for database write
        _save_to_db(order_id, amount, charge_result["charge_id"])

        span.set_status(Status(StatusCode.OK))
        span.add_event("payment.completed", {
            "payment.charge_id": charge_result["charge_id"],
        })
        return charge_result


def _charge_stripe(order_id: str, amount: float) -> dict:
    """Call Stripe API — child span example."""
    with tracer.start_as_current_span(
        "stripe.charge",
        kind=trace.SpanKind.CLIENT,
        attributes={
            SpanAttributes.RPC_SYSTEM: "http",
            SpanAttributes.NET_PEER_NAME: "api.stripe.com",
            "stripe.api_version": "2023-10-16",
            "order.id": order_id,
        }
    ) as span:
        start = time.monotonic()
        # ... actual Stripe call ...
        duration_ms = (time.monotonic() - start) * 1000
        span.set_attribute("stripe.duration_ms", duration_ms)
        span.set_attribute("stripe.charge_id", "ch_abc123")
        return {"charge_id": "ch_abc123", "status": "succeeded"}


def _save_to_db(order_id: str, amount: float, charge_id: str):
    """Database insert — child span example."""
    with tracer.start_as_current_span(
        "db.payments.insert",
        kind=trace.SpanKind.CLIENT,
        attributes={
            SpanAttributes.DB_SYSTEM: "postgresql",
            SpanAttributes.DB_NAME: "payments_db",
            SpanAttributes.DB_OPERATION: "INSERT",
            SpanAttributes.DB_SQL_TABLE: "payments",
            SpanAttributes.DB_STATEMENT: "INSERT INTO payments (order_id, amount, charge_id) VALUES (%s, %s, %s)",
        }
    ):
        pass  # execute query


if __name__ == "__main__":
    provider = init_tracer()
    try:
        result = process_payment("order-123", 49.99)
        print(f"Payment result: {result}")
    finally:
        provider.shutdown()
```

### Span Links

Span links associate a span with one or more other spans that are causally related but not in a parent-child relationship. Common use cases: batch processing (link each batch span to the originating request spans), async workflows, fan-in operations.

```go
// Go: creating span links
func ProcessBatch(ctx context.Context, requestContexts []context.Context) {
    // Collect links from all originating requests
    links := make([]trace.Link, 0, len(requestContexts))
    for _, reqCtx := range requestContexts {
        if span := trace.SpanFromContext(reqCtx); span.SpanContext().IsValid() {
            links = append(links, trace.Link{
                SpanContext: span.SpanContext(),
                Attributes: []attribute.KeyValue{
                    attribute.String("link.type", "originating_request"),
                },
            })
        }
    }

    // Create the batch span with links to all originating spans
    _, span := tracer.Start(ctx, "ProcessBatch",
        trace.WithLinks(links...),
        trace.WithSpanKind(trace.SpanKindConsumer),
        trace.WithAttributes(
            attribute.Int("batch.size", len(requestContexts)),
        ),
    )
    defer span.End()
    // ... process batch ...
}
```

```python
# Python: creating span links
from opentelemetry.trace import Link

def process_batch(contexts: list):
    links = [
        Link(
            context=trace.get_current_span(ctx).get_span_context(),
            attributes={"link.type": "originating_request"}
        )
        for ctx in contexts
        if trace.get_current_span(ctx).get_span_context().is_valid
    ]

    with tracer.start_as_current_span(
        "process_batch",
        links=links,
        attributes={"batch.size": len(contexts)}
    ) as span:
        # process batch...
        pass
```

---

## OTLP Collector Configuration

### Complete `otel-collector-config.yaml`

```yaml
# otel-collector-config.yaml
receivers:
  # OTLP receiver - accepts traces, metrics, logs
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        max_recv_msg_size_mib: 32
      http:
        endpoint: 0.0.0.0:4318
        cors:
          allowed_origins:
            - "http://localhost:*"
            - "https://*.mycompany.com"

  # Prometheus scraping (pull model)
  prometheus:
    config:
      scrape_configs:
        - job_name: 'otel-collector'
          scrape_interval: 15s
          static_configs:
            - targets: ['localhost:8888']
        - job_name: 'kubernetes-pods'
          kubernetes_sd_configs:
            - role: pod
          relabel_configs:
            - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
              action: keep
              regex: true

  # Jaeger receiver (legacy protocol support)
  jaeger:
    protocols:
      grpc:
        endpoint: 0.0.0.0:14250
      thrift_http:
        endpoint: 0.0.0.0:14268
      thrift_compact:
        endpoint: 0.0.0.0:6831
      thrift_binary:
        endpoint: 0.0.0.0:6832

  # Zipkin receiver
  zipkin:
    endpoint: 0.0.0.0:9411

  # Host metrics (CPU, memory, disk, network)
  hostmetrics:
    collection_interval: 30s
    scrapers:
      cpu:
      memory:
      disk:
      network:
      filesystem:

processors:
  # REQUIRED: Prevent out-of-memory crashes
  memory_limiter:
    check_interval: 1s
    limit_mib: 2048          # Hard limit: drop data above this
    spike_limit_mib: 512     # Soft limit: start GC above this

  # Batch spans/metrics/logs for efficiency
  batch:
    send_batch_size: 1024    # Max items per batch
    send_batch_max_size: 2048
    timeout: 5s              # Max time to wait before sending

  # Auto-detect resource attributes from the environment
  resourcedetection:
    detectors:
      - env              # OTEL_RESOURCE_ATTRIBUTES env var
      - system           # hostname, OS
      - docker           # container ID
      - gcp              # GCP project, zone, instance
      - aws              # AWS region, account, instance
      - azure            # Azure subscription, resource group
      - k8snode          # Kubernetes node attributes
    timeout: 10s
    override: false

  # Add, remove, or rename attributes
  attributes:
    actions:
      - key: deployment.environment
        value: production
        action: upsert
      - key: internal.secret
        action: delete         # Remove sensitive data

  # Add k8s metadata to spans from pod labels/annotations
  k8sattributes:
    auth_type: serviceAccount
    passthrough: false
    extract:
      metadata:
        - k8s.namespace.name
        - k8s.pod.name
        - k8s.pod.uid
        - k8s.node.name
        - k8s.deployment.name
        - k8s.container.name
      labels:
        - tag_name: app.label.version
          key: app.kubernetes.io/version
          from: pod

  # Tail-based sampling (makes sampling decisions AFTER seeing full trace)
  tail_sampling:
    decision_wait: 10s
    num_traces: 100000
    policies:
      - name: errors-policy
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow-traces-policy
        type: latency
        latency: {threshold_ms: 1000}
      - name: sample-10pct
        type: probabilistic
        probabilistic: {sampling_percentage: 10}

exporters:
  # Jaeger backend
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true

  # Prometheus exposition format (pull)
  prometheus:
    endpoint: "0.0.0.0:9464"
    namespace: otel
    send_timestamps: true
    metric_expiration: 5m
    resource_to_telemetry_conversion:
      enabled: true

  # OTLP to another collector (gateway pattern)
  otlp/gateway:
    endpoint: otel-gateway:4317
    tls:
      insecure: false
      ca_file: /certs/ca.crt
    headers:
      authorization: "Bearer ${env:OTEL_BEARER_TOKEN}"
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
      max_elapsed_time: 300s
    sending_queue:
      enabled: true
      num_consumers: 10
      queue_size: 1000

  # Elasticsearch / Elastic APM
  elasticsearch:
    endpoints: ["https://elasticsearch:9200"]
    index: otel-traces
    tls:
      ca_file: /certs/ca.crt

  # Debug / development (prints to stdout)
  debug:
    verbosity: detailed
    sampling_initial: 5
    sampling_thereafter: 200

extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  pprof:
    endpoint: 0.0.0.0:1777
  zpages:
    endpoint: 0.0.0.0:55679

service:
  extensions: [health_check, pprof, zpages]

  pipelines:
    traces:
      receivers: [otlp, jaeger, zipkin]
      processors: [memory_limiter, resourcedetection, k8sattributes, attributes, tail_sampling, batch]
      exporters: [otlp/jaeger, otlp/gateway, debug]

    metrics:
      receivers: [otlp, prometheus, hostmetrics]
      processors: [memory_limiter, resourcedetection, k8sattributes, batch]
      exporters: [prometheus, otlp/gateway]

    logs:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, k8sattributes, batch]
      exporters: [elasticsearch, otlp/gateway, debug]

  telemetry:
    logs:
      level: info
    metrics:
      address: 0.0.0.0:8888
```

### Docker Compose with Collector

```yaml
# docker-compose.yaml
version: "3.9"

services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.100.0
    command: ["--config=/etc/otelcol/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otelcol/otel-collector-config.yaml:ro
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "9411:9411"   # Zipkin
      - "14250:14250" # Jaeger gRPC
      - "14268:14268" # Jaeger HTTP
      - "9464:9464"   # Prometheus metrics endpoint
      - "13133:13133" # Health check
      - "55679:55679" # zPages debug UI
    environment:
      - OTEL_BEARER_TOKEN=${OTEL_BEARER_TOKEN}
    depends_on:
      - jaeger
    restart: unless-stopped

  jaeger:
    image: jaegertracing/all-in-one:1.57
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    ports:
      - "16686:16686" # Jaeger UI
      - "4317"        # Internal OTLP gRPC

  prometheus:
    image: prom/prometheus:v2.51.0
    volumes:
      - ./prometheus.yaml:/etc/prometheus/prometheus.yaml:ro
    command:
      - --config.file=/etc/prometheus/prometheus.yaml
      - --storage.tsdb.path=/prometheus
      - --web.enable-remote-write-receiver
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:10.4.0
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning

  # Example application
  payment-service:
    build: ./payment-service
    environment:
      - OTEL_SERVICE_NAME=payment-service
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
      - OTEL_TRACES_EXPORTER=otlp
      - OTEL_METRICS_EXPORTER=otlp
      - OTEL_LOGS_EXPORTER=otlp
    depends_on:
      - otel-collector
```

### Kubernetes ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: observability
data:
  otel-collector-config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      memory_limiter:
        check_interval: 1s
        limit_mib: 1500
      batch:
        timeout: 5s
      k8sattributes:
        auth_type: serviceAccount
        extract:
          metadata:
            - k8s.namespace.name
            - k8s.pod.name
            - k8s.deployment.name
            - k8s.node.name
    exporters:
      otlp/jaeger:
        endpoint: jaeger-collector.observability:4317
        tls:
          insecure: true
      prometheus:
        endpoint: "0.0.0.0:9464"
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, k8sattributes, batch]
          exporters: [otlp/jaeger]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [prometheus]
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otel-collector
  namespace: observability
spec:
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      serviceAccountName: otel-collector
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector-contrib:0.100.0
          args: ["--config=/conf/otel-collector-config.yaml"]
          ports:
            - containerPort: 4317  # OTLP gRPC
            - containerPort: 4318  # OTLP HTTP
            - containerPort: 9464  # Prometheus
          volumeMounts:
            - name: config
              mountPath: /conf
          resources:
            limits:
              cpu: "1"
              memory: "2Gi"
            requests:
              cpu: "200m"
              memory: "400Mi"
          livenessProbe:
            httpGet:
              path: /
              port: 13133
          readinessProbe:
            httpGet:
              path: /
              port: 13133
      volumes:
        - name: config
          configMap:
            name: otel-collector-config
```

---

## Exporters Configuration

### Jaeger Exporter

```yaml
# In otel-collector-config.yaml — Jaeger via OTLP
exporters:
  otlp/jaeger:
    endpoint: "jaeger-collector:4317"
    tls:
      insecure: true

# Legacy Jaeger exporter (deprecated, use OTLP instead)
  jaeger:
    endpoint: jaeger-collector:14250
    tls:
      insecure: true
```

```python
# Python SDK — send directly to Jaeger via OTLP
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
exporter = OTLPSpanExporter(endpoint="http://jaeger:4317", insecure=True)
```

### Zipkin Exporter

```yaml
# In otel-collector-config.yaml
exporters:
  zipkin:
    endpoint: "http://zipkin:9411/api/v2/spans"
    format: proto
    timeout: 10s
```

```python
# Python SDK — direct Zipkin export
from opentelemetry.exporter.zipkin.proto.http import ZipkinExporter
exporter = ZipkinExporter(endpoint="http://zipkin:9411/api/v2/spans")
```

### Prometheus Exporter

```yaml
# In otel-collector-config.yaml — expose /metrics endpoint
exporters:
  prometheus:
    endpoint: "0.0.0.0:9464"
    namespace: "myapp"
    resource_to_telemetry_conversion:
      enabled: true
    metric_expiration: 5m
```

```python
# Python SDK — Prometheus metric reader
from opentelemetry.exporter.prometheus import PrometheusExporter
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader

exporter = PrometheusExporter()
reader = PeriodicExportingMetricReader(exporter, export_interval_millis=60000)
provider = MeterProvider(metric_readers=[reader])
```

### OTLP/gRPC Exporter

```yaml
# In otel-collector-config.yaml — OTLP to another backend
exporters:
  otlp:
    endpoint: "https://api.honeycomb.io:443"
    headers:
      x-honeycomb-team: "${env:HONEYCOMB_API_KEY}"
    compression: gzip
    timeout: 10s
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
```

```go
// Go SDK — OTLP gRPC exporter with TLS
import "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
import "google.golang.org/grpc/credentials"

creds, _ := credentials.NewClientTLSFromFile("ca.crt", "")
exporter, _ := otlptracegrpc.New(ctx,
    otlptracegrpc.WithEndpoint("otel-collector:4317"),
    otlptracegrpc.WithTLSCredentials(creds),
    otlptracegrpc.WithHeaders(map[string]string{
        "Authorization": "Bearer " + apiKey,
    }),
)
```

### Console/Logging Exporter (for debugging)

```python
# Python SDK — print spans to stdout
from opentelemetry.sdk.trace.export import ConsoleSpanExporter, SimpleSpanProcessor

provider = TracerProvider()
provider.add_span_processor(
    SimpleSpanProcessor(ConsoleSpanExporter())
)
trace.set_tracer_provider(provider)
```

```go
// Go SDK — stdout exporter
import "go.opentelemetry.io/otel/exporters/stdout/stdouttrace"

exporter, _ := stdouttrace.New(
    stdouttrace.WithPrettyPrint(),
)
```

---

## Semantic Conventions

### HTTP Conventions

| Attribute | Type | Example | Required |
|-----------|------|---------|----------|
| `http.request.method` | string | `GET`, `POST` | Required |
| `http.response.status_code` | int | `200`, `404` | Required |
| `url.scheme` | string | `https` | Required |
| `url.path` | string | `/api/v1/payments` | Recommended |
| `url.query` | string | `?page=1` | Optional |
| `server.address` | string | `api.example.com` | Recommended |
| `server.port` | int | `443` | Recommended |
| `http.request.body.size` | int | `1234` | Optional |
| `http.response.body.size` | int | `5678` | Optional |
| `user_agent.original` | string | `Mozilla/5.0 ...` | Recommended |
| `network.protocol.version` | string | `1.1`, `2` | Recommended |
| `client.address` | string | `192.168.1.1` | Optional |

### Database Conventions

| Attribute | Type | Example | Required |
|-----------|------|---------|----------|
| `db.system` | string | `postgresql`, `mysql`, `redis` | Required |
| `db.name` | string | `payments_db` | Recommended |
| `db.operation` | string | `SELECT`, `INSERT` | Recommended |
| `db.sql.table` | string | `payments` | Recommended |
| `db.statement` | string | `SELECT * FROM payments WHERE id=?` | Optional |
| `db.connection_string` | string | `Server=localhost;Database=mydb` | Optional |
| `db.user` | string | `app_user` | Optional |
| `server.address` | string | `db.example.com` | Recommended |
| `server.port` | int | `5432` | Recommended |
| `db.redis.database_index` | int | `0` | Conditional |
| `db.mongodb.collection` | string | `users` | Conditional |

### Messaging Conventions

| Attribute | Type | Example | Required |
|-----------|------|---------|----------|
| `messaging.system` | string | `kafka`, `rabbitmq`, `sqs` | Required |
| `messaging.operation` | string | `publish`, `receive`, `process` | Required |
| `messaging.destination.name` | string | `payment-events` | Required |
| `messaging.destination.kind` | string | `topic`, `queue` | Recommended |
| `messaging.message.id` | string | `msg-abc-123` | Recommended |
| `messaging.batch.message_count` | int | `50` | Conditional |
| `messaging.kafka.consumer.group` | string | `payment-consumers` | Conditional |
| `messaging.kafka.partition` | int | `3` | Optional |
| `messaging.kafka.offset` | int | `12345` | Optional |
| `messaging.rabbitmq.destination.routing_key` | string | `payment.created` | Conditional |

### General Resource Attributes

| Attribute | Type | Example | Description |
|-----------|------|---------|-------------|
| `service.name` | string | `payment-service` | Logical service name |
| `service.version` | string | `1.2.3` | Service version |
| `service.namespace` | string | `com.mycompany` | Service namespace |
| `service.instance.id` | string | `pod-abc-123` | Unique instance ID |
| `deployment.environment` | string | `production` | Deployment environment |
| `host.name` | string | `server-01` | Hostname |
| `host.id` | string | `i-1234567890abcdef0` | Host ID (e.g., EC2 instance ID) |
| `os.type` | string | `linux` | Operating system type |
| `process.pid` | int | `12345` | Process ID |
| `process.runtime.name` | string | `CPython` | Runtime name |
| `k8s.pod.name` | string | `payment-svc-abc-xyz` | Kubernetes pod name |
| `k8s.namespace.name` | string | `production` | Kubernetes namespace |
| `k8s.node.name` | string | `worker-1` | Kubernetes node name |
| `cloud.provider` | string | `aws`, `gcp`, `azure` | Cloud provider |
| `cloud.region` | string | `us-east-1` | Cloud region |

---

## Propagators

Propagators inject and extract context (trace ID, span ID, baggage) from carrier formats like HTTP headers, so distributed traces work across service boundaries.

```
Context Propagation Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 Service A                     HTTP Request              Service B
 ┌─────────────────────┐                                ┌─────────────────────┐
 │ span = start_span() │                                │                     │
 │                     │  inject(context, headers)      │  ctx = extract(     │
 │ propagator.inject() │──────────────────────────────▶ │    headers)         │
 │                     │  traceparent: 00-abc123-def456  │                     │
 │                     │  tracestate:  vendor-specific  │  child = start_span │
 │                     │  baggage:     user-id=42        │    (parent=ctx)     │
 └─────────────────────┘                                └─────────────────────┘
```

### W3C TraceContext (default)

The W3C TraceContext is the **IETF standard** and the **default OTel propagator**. It uses two HTTP headers:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^ ^^
             |   traceId (128-bit hex)            spanId (64-bit)  flags
             version (00)                                          (01=sampled)

tracestate: rojo=00f067aa0ba902b7,congo=t61rcWkgMzE
            ^vendor-specific key=value pairs^
```

### B3 Single and Multi-Header

B3 is used by Zipkin and older systems. OTel supports both formats:

```
# B3 Single header (compact)
b3: {traceId}-{spanId}-{sampling}-{parentSpanId}
b3: 80f198ee56343ba864fe8b2a57d3eff7-e457b5a2e4d86bd1-1-05e3ac9a4f6e3b90

# B3 Multi-header (original Zipkin format)
X-B3-TraceId: 80f198ee56343ba864fe8b2a57d3eff7
X-B3-SpanId: e457b5a2e4d86bd1
X-B3-ParentSpanId: 05e3ac9a4f6e3b90
X-B3-Sampled: 1
```

### Baggage Propagator

Baggage carries user-defined key-value pairs across service boundaries. Useful for passing user ID, tenant ID, or feature flags to all downstream services.

```
baggage: user-id=12345,tenant-id=acme-corp,feature-flag=new-checkout
```

**Important:** Baggage is sent in every request — keep it small. Do not put sensitive data in baggage.

### Configuring Propagators

```python
# Python — configure multiple propagators
from opentelemetry import propagate
from opentelemetry.propagators.composite import CompositePropagator
from opentelemetry.propagators.b3 import B3SingleFormat, B3MultiFormat
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
from opentelemetry.baggage.propagation import W3CBaggagePropagator

# Use W3C TraceContext + Baggage (default)
propagate.set_global_textmap(
    CompositePropagator([
        TraceContextTextMapPropagator(),
        W3CBaggagePropagator(),
    ])
)

# Or add B3 for backward compatibility with Zipkin
propagate.set_global_textmap(
    CompositePropagator([
        TraceContextTextMapPropagator(),
        W3CBaggagePropagator(),
        B3MultiFormat(),
    ])
)
```

```go
// Go — configure propagators at startup
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/contrib/propagators/b3"
)

// W3C TraceContext + Baggage (recommended)
otel.SetTextMapPropagator(
    propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ),
)

// Adding B3 support
otel.SetTextMapPropagator(
    propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
        b3.New(b3.WithInjectEncoding(b3.B3MultipleHeader)),
    ),
)
```

```bash
# Via environment variable
export OTEL_PROPAGATORS=tracecontext,baggage         # Default
export OTEL_PROPAGATORS=tracecontext,baggage,b3multi # With B3
export OTEL_PROPAGATORS=tracecontext,baggage,b3      # With B3 single header
```

---

## OTel Demo Architecture

The [OpenTelemetry Demo](https://github.com/open-telemetry/opentelemetry-demo) is an official reference application demonstrating OTel across multiple languages and services, modeled as an e-commerce platform.

```
OTel Demo Service Map
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

                           ┌───────────────┐
                           │   Frontend    │ (Next.js, TypeScript)
                           │   (port 8080) │
                           └───────┬───────┘
                                   │ HTTP
                           ┌───────▼───────┐
                           │  Frontend Proxy│ (Envoy)
                           └───────┬───────┘
                                   │
              ┌────────────────────┼─────────────────────┐
              │                    │                      │
    ┌─────────▼──────┐   ┌────────▼────────┐   ┌────────▼────────┐
    │ Product Catalog │   │   Cart Service  │   │Checkout Service │
    │   (Go)          │   │   (Dotnet)      │   │   (Go)          │
    └────────────────┘   └────────┬────────┘   └────────┬────────┘
                                  │ Redis                │
                         ┌────────▼────────┐   ┌────────▼────────┐
                         │  Redis Cache    │   │ Payment Service │
                         └────────────────┘   │   (JavaScript)  │
                                               └────────┬────────┘
    ┌───────────────────┐                               │
    │  Recommendation   │◀────── Product Catalog        │
    │  Service (Python) │                      ┌────────▼────────┐
    └───────────────────┘                      │  Email Service  │
                                               │   (Ruby)        │
    ┌───────────────────┐                      └────────────────┘
    │  Currency Service │
    │   (C++)           │  ┌─────────────────────────────────────┐
    └───────────────────┘  │           Observability Stack       │
                           │  OTel Collector → Jaeger            │
    ┌───────────────────┐  │               → Prometheus          │
    │  Shipping Service │  │               → Grafana             │
    │   (Rust)          │  │               → OpenSearch          │
    └───────────────────┘  └─────────────────────────────────────┘
    ┌───────────────────┐
    │   Ad Service      │  All services send telemetry to the
    │   (Java)          │  OTel Collector via OTLP gRPC.
    └───────────────────┘  The Collector fans out to all backends.
    ┌───────────────────┐
    │  Load Generator   │  Languages covered: Go, Java, Python,
    │  (Python/Locust)  │  JavaScript, TypeScript, Dotnet,
    └───────────────────┘  Rust, C++, PHP, Ruby
```

**Key demo features:**
- Every service uses the OTel API in its native language
- Auto-instrumentation used where available; manual instrumentation for business logic
- Demonstrates baggage propagation across all services
- Includes Grafana dashboards, Jaeger traces, and Prometheus metrics pre-wired
- Includes feature flag service to trigger errors for demo purposes

---

## Migration Guide

### From OpenTracing

OpenTracing provided only a tracing API. OTel replaces all OpenTracing concepts with equivalents.

| OpenTracing Concept | OTel Equivalent | Notes |
|--------------------|----------------|-------|
| `Tracer` | `trace.Tracer` | Get via `otel.Tracer("name")` |
| `Span` | `trace.Span` | Same concept, more attributes |
| `SpanContext` | `trace.SpanContext` | Includes `TraceFlags`, `TraceState` |
| `Baggage` | `baggage.Baggage` | Separated from span context |
| `Tags` | `span.SetAttributes(...)` | Use semantic conventions |
| `Logs` (span logs) | `span.AddEvent(...)` | Timestamped events with attributes |
| `References` (ChildOf) | Parent span context | Automatic via context propagation |
| `References` (FollowsFrom) | `trace.Link` | Use `WithLinks(...)` option |
| `Inject/Extract` | `propagator.Inject/Extract` | W3C TraceContext by default |
| `GlobalTracer` | `otel.GetTracerProvider()` | Global provider |
| `StartSpanFromContext` | `tracer.Start(ctx, name)` | Returns `(ctx, span)` |
| `FinishSpan` | `span.End()` | Call deferred |

**Code comparison:**
```go
// OpenTracing (old)
span, ctx := opentracing.StartSpanFromContext(ctx, "ProcessPayment")
span.SetTag("order.id", orderID)
span.SetTag("error", true)
span.LogFields(log.String("event", "payment.failed"), log.Error(err))
defer span.Finish()

// OpenTelemetry (new)
ctx, span := tracer.Start(ctx, "ProcessPayment",
    trace.WithAttributes(attribute.String("order.id", orderID)),
)
span.RecordError(err)
span.SetStatus(codes.Error, "payment failed")
defer span.End()
```

```python
# OpenTracing (old)
with opentracing.tracer.start_active_span("ProcessPayment") as scope:
    scope.span.set_tag("order.id", order_id)
    scope.span.set_tag(tags.HTTP_STATUS_CODE, 500)
    scope.span.log_kv({"event": "error", "message": str(err)})

# OpenTelemetry (new)
with tracer.start_as_current_span("ProcessPayment") as span:
    span.set_attribute("order.id", order_id)
    span.set_attribute(SpanAttributes.HTTP_STATUS_CODE, 500)
    span.record_exception(err)
    span.set_status(Status(StatusCode.ERROR, str(err)))
```

**OTel provides shim libraries for zero-code migration:**
```go
// Install the shim — existing OpenTracing code continues working
import otbridge "go.opentelemetry.io/otel/bridge/opentracing"

// Wrap OTel tracer provider as an OpenTracing tracer
tp := otel.GetTracerProvider()
bridgeTracer, wrapperProvider := otbridge.NewTracerPair(tp.Tracer("bridge"))
opentracing.SetGlobalTracer(bridgeTracer)
// Now OpenTracing API calls are backed by OTel
```

### From OpenCensus

| OpenCensus Concept | OTel Equivalent | Notes |
|-------------------|----------------|-------|
| `trace.StartSpan` | `tracer.Start(ctx, name)` | Same pattern |
| `span.Annotate` | `span.AddEvent` | Events with attributes |
| `span.AddAttributes` | `span.SetAttributes` | Semantic conventions available |
| `span.End` | `span.End` | Identical |
| `stats.Record` | `meter.Record / Add` | Different API |
| `stats.Int64Measure` | `meter.Int64Counter` (or histogram) | Type changes |
| `stats.Float64Measure` | `meter.Float64Histogram` | Type changes |
| `view.Register` | `PeriodicExportingMetricReader` | Reader-based |
| `tag.NewMap` | `attribute.NewSet` | Moved to attributes |
| `tag.Key` | `attribute.Key` | Renamed |
| `ochttp.Handler` | OTel HTTP instrumentation | Drop-in library |
| `ocgrpc.Handler` | OTel gRPC instrumentation | Drop-in library |

**OpenCensus shim for Go:**
```go
// The shim routes OpenCensus spans into OTel pipeline
import ocbridge "go.opentelemetry.io/otel/bridge/opencensus"

// Install the OC handler
ocbridge.InstallTraceBridge()
// Existing OpenCensus instrumentation now flows through OTel
```

---

## Next Steps

1. **Deploy the OTel Collector** — Start with the `otel-collector-config.yaml` in this guide and deploy alongside your application or as a DaemonSet in Kubernetes.
2. **Auto-instrument your services** — Use the language-specific auto-instrumentation agent for your primary language; add manual instrumentation for business-critical operations.
3. **Implement Semantic Conventions** — Review the HTTP, database, and messaging conventions and ensure your spans use standard attribute names.
4. **Configure tail-based sampling** — Use the `tail_sampling` processor to keep 100% of error traces and slow traces while sampling healthy traffic.
5. **Set up the OTel Demo** — Run `docker compose up` in the [opentelemetry-demo](https://github.com/open-telemetry/opentelemetry-demo) repository for a hands-on reference.
6. **Migrate from OpenTracing/OpenCensus** — Use the bridge shim libraries for a gradual, zero-downtime migration.
7. **Explore Profiles** — The experimental Profiling signal in OTel bridges APM traces to continuous profiling data for correlated analysis.

**Related guides in this series:**
- [03-DISTRIBUTED-TRACING.md](03-DISTRIBUTED-TRACING.md) — Jaeger, Zipkin, sampling strategies
- [05-ALERTING.md](05-ALERTING.md) — Alerting on OTel-derived metrics
- [06-APM.md](06-APM.md) — Commercial APM tools that accept OTLP

---

## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-01-01 | Platform Engineering | Initial document |
| 1.1.0 | 2025-01-15 | Platform Engineering | Added OTel Operator Kubernetes example |
| 1.2.0 | 2025-02-01 | Platform Engineering | Added Node.js auto-instrumentation section |
| 1.3.0 | 2025-03-01 | Platform Engineering | Added migration guides from OpenTracing/OpenCensus |
| 1.4.0 | 2025-06-01 | Platform Engineering | Updated collector config to contrib 0.100.0 |
