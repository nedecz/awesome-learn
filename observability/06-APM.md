# Application Performance Monitoring (APM)

## Table of Contents

1. [What is APM](#what-is-apm)
2. [APM Capabilities](#apm-capabilities)
3. [Datadog APM](#datadog-apm)
4. [New Relic](#new-relic)
5. [Dynatrace](#dynatrace)
6. [Elastic APM](#elastic-apm)
7. [Continuous Profiling](#continuous-profiling)
8. [Pyroscope](#pyroscope)
9. [eBPF-based Profiling](#ebpf-based-profiling)
10. [Performance Baselines and Anomaly Detection](#performance-baselines-and-anomaly-detection)
11. [APM Comparison Table](#apm-comparison-table)
12. [Cost Considerations](#cost-considerations)
13. [Next Steps](#next-steps)
14. [Version History](#version-history)

---

## What is APM

**Application Performance Monitoring (APM)** is the practice of monitoring and managing the performance and availability of software applications. APM tools collect, correlate, and visualize data about how applications behave in production, enabling engineers to quickly identify, diagnose, and resolve performance problems before they impact users.

### How APM Differs from Basic Metrics Monitoring

| Dimension | Basic Metrics Monitoring | APM |
|-----------|------------------------|-----|
| Granularity | Service-level aggregates (CPU, memory, request rate) | Individual transaction traces, function-level timing |
| Correlation | Metrics exist in isolation | Traces link metrics to specific code paths |
| Code context | No visibility into application internals | Stack traces, slow method identification, DB queries |
| Error detail | Error count/rate | Full stack trace, parameters, user context |
| Dependency mapping | Manual or infrastructure-level | Automatic service dependency discovery |
| Profiling | None | CPU/memory profiles linked to traces |
| User context | Not typically available | RUM links frontend user actions to backend traces |

### APM Capabilities Overview

```
APM Data Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  User Action
      │
      ▼
  ┌─────────────┐    Real User Monitoring (RUM)
  │  Browser /  │───▶ Page load time, JS errors, user journeys
  │  Mobile App │
  └──────┬──────┘
         │ HTTP Request
         ▼
  ┌─────────────┐    APM Traces
  │  Service A  │───▶ Trace ID, span timing, method-level detail
  │  (Go / Java)│    Error context, DB queries, external calls
  └──────┬──────┘
         │ gRPC / HTTP
         ▼
  ┌─────────────┐    Service Map
  │  Service B  │───▶ Automatic dependency graph
  │  (Python)   │    Throughput, error rate, latency per edge
  └──────┬──────┘
         │ SQL
         ▼
  ┌─────────────┐    Database Monitoring
  │  PostgreSQL │───▶ Slow query log, EXPLAIN plans, table scans
  └─────────────┘

  Across all services:
  ┌─────────────────────────────────────────────────────────┐
  │  Continuous Profiling → CPU flame graphs per trace      │
  │  Error Tracking       → Grouped exceptions, regressions │
  │  Synthetic Monitoring → External health checks          │
  └─────────────────────────────────────────────────────────┘
```

---

## APM Capabilities

### Transaction Tracing

Transaction tracing follows a single user request from entry point (browser/mobile) through every service, database call, cache lookup, and external API call it touches. Each unit of work is a **span**; all spans for one request form a **trace**.

```
Transaction Trace — Checkout Flow
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  frontend.checkout                        [████████████████████] 820ms
    api-gateway.route                       [████████████████████] 815ms
      order-service.createOrder              [███████████████] 680ms
        db.orders.INSERT                      [███] 45ms
        inventory-service.checkStock          [████] 60ms
          db.inventory.SELECT                  [██] 30ms
        payment-service.charge                [██████████] 520ms
          stripe.api.createCharge              [█████████] 490ms  ← SLOW
        notification-service.sendEmail        [█] 15ms
```

### Dependency / Service Map

APM automatically builds a **service dependency map** by analyzing trace data — showing which services call which, with throughput, error rate, and latency metrics on each edge.

```
Service Map
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

            ┌──────────────┐
            │   Frontend   │ 1200 req/min, 0.1% err
            └──────┬───────┘
                   │
                   ▼
            ┌──────────────┐
            │  API Gateway │ 1200 req/min, 0.2% err
            └──────┬───────┘
         ┌─────────┼──────────┐
         ▼         ▼          ▼
    ┌─────────┐ ┌────────┐ ┌────────────┐
    │  Order  │ │  User  │ │  Product   │
    │ Service │ │Service │ │  Catalog   │
    │200r/m   │ │800r/m  │ │ 400r/m     │
    │0.5%err  │ │0.0%err │ │ 0.0%err    │
    └────┬────┘ └────┬───┘ └─────┬──────┘
         │           │           │
         ▼           ▼           ▼
    ┌────────┐  ┌─────────┐  ┌────────┐
    │Payment │  │  Auth   │  │  Redis │
    │Service │  │  Cache  │  │ Cache  │
    │0.8%err │  │0.0%err  │  │0.0%err │
    └────────┘  └─────────┘  └────────┘
```

### Error Tracking and Grouping

APM groups similar errors together (by stack trace fingerprint), tracks their occurrence over time, assigns them to commits/deployments, and surfaces regressions when error counts spike after a deploy.

### Continuous Profiling

Continuous profiling samples stack traces from production code at low overhead (typically 1–5% CPU overhead), collecting statistical data about which functions consume the most CPU time, memory, or I/O. Unlike sampling in development, it runs 24/7 in production.

### Real User Monitoring (RUM)

RUM collects data from real user browsers or mobile apps: page load time, JavaScript errors, Core Web Vitals (LCP, FID, CLS), user journeys, and geographic performance differences.

### Synthetic Monitoring

Synthetic monitoring runs automated scripts (from distributed locations globally) that simulate user interactions — login, search, checkout — and alert when they fail or become slow. Works even when no real users are active.

---

## Datadog APM

### Setup

**Docker Compose with Datadog Agent:**
```yaml
version: "3.9"
services:
  datadog-agent:
    image: datadog/agent:7
    environment:
      DD_API_KEY: "${DD_API_KEY}"
      DD_SITE: "datadoghq.com"          # or datadoghq.eu for EU
      DD_APM_ENABLED: "true"
      DD_APM_NON_LOCAL_TRAFFIC: "true"  # Accept traces from other containers
      DD_DOGSTATSD_NON_LOCAL_TRAFFIC: "true"
      DD_LOGS_ENABLED: "true"
      DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL: "true"
      DD_PROCESS_AGENT_ENABLED: "true"
      DD_CONTAINER_EXCLUDE: "name:datadog-agent"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /proc/:/host/proc/:ro
      - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro
    ports:
      - "8125:8125/udp"   # DogStatsD
      - "8126:8126/tcp"   # APM trace intake

  payment-service:
    image: myregistry/payment-service:latest
    environment:
      DD_AGENT_HOST: datadog-agent
      DD_TRACE_AGENT_PORT: "8126"
      DD_SERVICE: "payment-service"
      DD_ENV: "production"
      DD_VERSION: "1.2.3"
      DD_LOGS_INJECTION: "true"         # Correlate logs with traces
      DD_RUNTIME_METRICS_ENABLED: "true"
    depends_on:
      - datadog-agent
```

**Python agent installation:**
```bash
pip install ddtrace

# Run with auto-instrumentation
DD_SERVICE=payment-service \
DD_ENV=production \
DD_VERSION=1.2.3 \
DD_LOGS_INJECTION=true \
ddtrace-run python app.py
```

**Java agent:**
```bash
# Download agent
curl -Lo dd-java-agent.jar \
  https://dtdg.co/latest-java-tracer

# Run with agent
java -javaagent:dd-java-agent.jar \
     -Ddd.service=payment-service \
     -Ddd.env=production \
     -Ddd.version=1.2.3 \
     -Ddd.logs.injection=true \
     -jar myapp.jar
```

**Go agent:**
```bash
go get gopkg.in/DataDog/dd-trace-go.v1/ddtrace
```

```go
import (
    "gopkg.in/DataDog/dd-trace-go.v1/ddtrace/tracer"
    httptrace "gopkg.in/DataDog/dd-trace-go.v1/contrib/net/http"
)

func main() {
    tracer.Start(
        tracer.WithService("payment-service"),
        tracer.WithEnv("production"),
        tracer.WithServiceVersion("1.2.3"),
    )
    defer tracer.Stop()

    // Auto-instrument HTTP router
    mux := httptrace.NewServeMux()
    mux.HandleFunc("/payments", handlePayment)
}
```

### Key Features

| Feature | Description |
|---------|-------------|
| Trace Search & Analytics | Full-text search over 100% of ingested traces with custom retention |
| Service Maps | Auto-generated dependency graph with health metrics on edges |
| Flame Graphs | In-trace flame graphs showing method-level CPU breakdown |
| Watchdog AI | ML-based anomaly detection that surfaces p-value anomalies automatically |
| Error Tracking | Grouped errors with regression detection on deploy |
| Deployment Tracking | Correlate deployments with performance changes automatically |
| Database Monitoring | Query-level performance with explain plans and WAIT event analysis |
| Profiling | Always-on profiling linked to traces |

### Datadog Query Examples

```python
# Custom metrics via DogStatsD
from datadog import statsd

# Counter
statsd.increment("payment.processed", tags=["status:success", "currency:usd"])

# Histogram (timing)
statsd.histogram("payment.duration", duration_ms, tags=["processor:stripe"])

# Gauge
statsd.gauge("payment.queue.depth", queue_size)
```

```python
# Datadog API queries (equivalent to NRQL)
# In the Metrics Explorer or dashboards, use:

# P99 latency by service
avg:trace.web.request.duration.by.service{env:production} by {service}.rollup(p99, 60)

# Error rate
sum:trace.web.request.errors{env:production} by {service} /
sum:trace.web.request.hits{env:production} by {service} * 100

# Apdex score (satisfied < 0.5s, tolerating < 2s)
(
  sum:trace.web.request.apdex.by.service{env:production, apdex_type:satisfied} +
  sum:trace.web.request.apdex.by.service{env:production, apdex_type:tolerating} / 2
) / sum:trace.web.request.apdex.by.service{env:production} by {service}
```

### Datadog Monitor (Alert) Configuration

```yaml
# datadog-monitors.yaml (using Terraform or API)
resource "datadog_monitor" "high_error_rate" {
  name    = "High Error Rate - {{ service }}"
  type    = "metric alert"
  message = <<-EOT
    Error rate for {{service.name}} is {{value}}%.
    Runbook: https://wiki.example.com/runbooks/high-error-rate
    @pagerduty-critical
  EOT

  query = <<-EOQ
    sum(last_5m):
      sum:trace.web.request.errors{env:production} by {service}.as_rate() /
      sum:trace.web.request.hits{env:production} by {service}.as_rate() * 100
    > 1.0
  EOQ

  thresholds = {
    critical = 1.0
    warning  = 0.5
  }

  notify_no_data    = false
  evaluation_delay  = 60
  renotify_interval = 60

  tags = ["env:production", "team:platform", "severity:critical"]
}
```

---

## New Relic

### Setup

**Java agent:**
```bash
# Download
curl -O https://download.newrelic.com/newrelic/java-agent/newrelic-agent/current/newrelic-java.zip
unzip newrelic-java.zip

# Configure newrelic.yml
cat > newrelic/newrelic.yml << 'EOF'
common: &default_settings
  license_key: ${NEW_RELIC_LICENSE_KEY}
  app_name: payment-service
  distributed_tracing:
    enabled: true
  log_file_name: STDOUT
  audit_mode: false
  transaction_tracer:
    enabled: true
    transaction_threshold: 0.5s     # Trace requests > 500ms
    record_sql: obfuscated
    stack_trace_threshold: 0.5s
  error_collector:
    enabled: true
    ignore_status_codes: 404
  browser_monitoring:
    auto_instrument: true
EOF

# Run with agent
java -javaagent:newrelic/newrelic.jar \
     -Dnewrelic.config.file=newrelic/newrelic.yml \
     -jar myapp.jar
```

**Node.js agent:**
```bash
npm install newrelic
```

```javascript
// MUST be the first require in your app
require('newrelic');

// Or configure via environment variables:
// NEW_RELIC_LICENSE_KEY=...
// NEW_RELIC_APP_NAME=payment-service
// NEW_RELIC_DISTRIBUTED_TRACING_ENABLED=true
// NEW_RELIC_LOG=stdout
```

### Distributed Tracing Setup

```javascript
// newrelic.js configuration file
'use strict';

exports.config = {
  app_name: ['payment-service'],
  license_key: process.env.NEW_RELIC_LICENSE_KEY,
  distributed_tracing: {
    enabled: true,
  },
  transaction_tracer: {
    enabled: true,
    transaction_threshold: 0.5,   // seconds
    record_sql: 'obfuscated',
    explain_threshold: 0.5,
  },
  error_collector: {
    enabled: true,
    ignore_status_codes: [404],
  },
  logging: {
    level: 'info',
    filepath: 'stdout',
    enabled: true,
  },
};
```

### NRQL Examples

```sql
-- Response time percentiles by service
SELECT percentile(duration, 50, 90, 99) 
FROM Transaction 
WHERE appName = 'payment-service' 
SINCE 1 hour ago 
FACET name

-- Error rate over time
SELECT 
  count(*) AS total,
  filter(count(*), WHERE error IS true) AS errors,
  percentage(count(*), WHERE error IS true) AS errorRate
FROM Transaction 
WHERE appName = 'payment-service'
SINCE 30 minutes ago 
TIMESERIES 1 minute

-- Throughput (requests per minute)
SELECT rate(count(*), 1 minute) AS rpm
FROM Transaction 
WHERE appName = 'payment-service'
SINCE 1 hour ago 
TIMESERIES 1 minute

-- Apdex score (T = 0.5s)
SELECT apdex(duration, 0.5) AS apdexScore
FROM Transaction 
WHERE appName = 'payment-service'
SINCE 1 hour ago 
FACET appName

-- Top slowest transactions
SELECT average(duration) AS avgDuration, count(*) AS callCount
FROM Transaction
WHERE appName = 'payment-service'
SINCE 1 hour ago
FACET name
ORDER BY avgDuration DESC
LIMIT 20

-- Slow database queries
SELECT average(duration) AS avgDuration, 
       count(*) AS executions,
       average(databaseDuration) AS dbTime
FROM Transaction
WHERE appName = 'payment-service' 
  AND databaseDuration > 0.1
SINCE 1 hour ago
FACET databaseCallCount, name
ORDER BY dbTime DESC
LIMIT 10

-- Service dependency map data
SELECT count(*) AS callCount, 
       average(externalDuration) AS avgLatency
FROM Transaction 
WHERE appName = 'payment-service'
SINCE 1 hour ago
FACET externalHost

-- Error analysis by error message
SELECT count(*) AS occurrences,
       latest(errorMessage) AS lastMessage
FROM TransactionError
WHERE appName = 'payment-service'
SINCE 24 hours ago
FACET errorClass
ORDER BY occurrences DESC
```

---

## Dynatrace

### OneAgent Deployment

**Kubernetes Operator:**
```bash
# Install the Dynatrace Operator
kubectl create namespace dynatrace
kubectl apply -f https://github.com/Dynatrace/dynatrace-operator/releases/latest/download/kubernetes.yaml

# Create the DynaKube custom resource
cat <<EOF | kubectl apply -f -
apiVersion: dynatrace.com/v1beta1
kind: DynaKube
metadata:
  name: dynakube
  namespace: dynatrace
spec:
  apiUrl: https://ENVIRONMENT_ID.live.dynatrace.com/api
  tokens: dynatrace-tokens
  oneAgent:
    classicFullStack:
      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/master
          operator: Exists
  activeGate:
    capabilities:
      - routing
      - kubernetes-monitoring
      - dynatrace-api
EOF
```

**Docker:**
```bash
docker run -d \
  --name oneagent \
  --privileged \
  --pid=host \
  --net=host \
  -v /:/mnt/root \
  -e ONEAGENT_INSTALLER_SCRIPT_URL="https://ENVIRONMENT_ID.live.dynatrace.com/api/v1/deployment/installer/agent/unix/default/latest?arch=x86&flavor=default" \
  -e ONEAGENT_INSTALLER_DOWNLOAD_TOKEN="${DYNATRACE_PAAS_TOKEN}" \
  dynatrace/oneagent:latest
```

### PurePath Technology

**PurePath** is Dynatrace's approach to distributed tracing. Unlike sampling-based tracers, PurePath captures **every single transaction** end-to-end by injecting sensors directly into code at the bytecode level (for JVM/CLR) or via hooks (for Node.js, Python).

```
PurePath Data Model
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  User Action (browser click)
      │
      ▼ (RUM sensor injected in HTML)
  Browser PurePath
      │
      ▼ (HTTP header injection, automatic)
  Frontend Service PurePath
      │ (every method in the call chain is timed, no sampling)
      ├── Method: handleRequest() [12ms]
      ├──── Method: validateInput() [2ms]
      ├──── Method: callBackend() [8ms]
      │         │
      │         ▼ (outbound HTTP call detected automatically)
      │    Backend Service PurePath
      │         │
      │         ├── DB call: SELECT * FROM payments [4ms]
      │         └── Cache hit: Redis GET [0.5ms]
      └── Method: renderResponse() [2ms]

  Every node in the tree: timing, CPU samples, memory alloc, errors
```

### Smartscape Topology

**Smartscape** is Dynatrace's real-time topology map. It automatically discovers and visualizes the full stack — from datacenter/cloud provider down to individual processes and their dependencies — updated continuously from OneAgent data.

```
Smartscape Layers
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Datacenter / Cloud   →  AWS us-east-1 VPC
  Host                 →  EC2 instances, Kubernetes nodes
  Process group        →  payment-service (all instances)
  Service              →  HTTP endpoint group (logical)
  Application          →  checkout-app (user-facing)
```

### Davis AI Anomaly Detection

**Davis AI** is Dynatrace's automated root cause analysis engine. It ingests all metric, trace, log, topology, and event data and automatically:
1. Detects anomalies using learned baselines (no static thresholds needed)
2. Correlates anomalies across services and layers
3. Determines root cause by analyzing the causal chain
4. Creates a **Problem** entity with root cause and affected services
5. Sends a single alert per problem (not per symptom)

### DQL Query Examples

```dql
// Dynatrace Query Language (DQL) — used in Notebooks and Dashboards

// P99 response time by service over last 2 hours
fetch spans
| filter dt.entity.service == "payment-service"
| filter duration > 0
| summarize p99 = percentile(duration, 99), count = count() by bin(timestamp, 5m), service.name
| sort timestamp desc

// Error rate calculation
fetch spans
| filter isNotNull(error.message)
| summarize errors = count() by bin(timestamp, 1m), service.name
| join kind=inner (
    fetch spans
    | summarize total = count() by bin(timestamp, 1m), service.name
  ), on: timestamp, service.name
| fieldsAdd errorRate = errors / total * 100

// Slow database queries
fetch spans
| filter span.kind == "client"
| filter db.system != ""
| filter duration > 500ms
| fields timestamp, service.name, db.system, db.statement, duration
| sort duration desc
| limit 20

// Top endpoints by traffic
fetch spans
| filter span.kind == "server"
| summarize callCount = count(), avgDuration = avg(duration) by http.url, service.name
| sort callCount desc
| limit 10
```

---

## Elastic APM

### Architecture

```
Elastic APM Architecture
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Application Services
  ┌────────────────────────────────────────────────────────────┐
  │  Java Agent  │  Python Agent  │  Node.js Agent  │  Go SDK  │
  └──────┬───────┴───────┬────────┴────────┬─────────┴────┬────┘
         │               │                 │              │
         └───────────────┼─────────────────┼──────────────┘
                         │ HTTP (port 8200)
                         ▼
                  ┌─────────────┐
                  │  APM Server │  Receives traces, metrics, logs
                  │             │  Transforms to Elasticsearch format
                  └──────┬──────┘
                         │
                         ▼
                  ┌─────────────┐
                  │Elasticsearch│  Stores all APM data
                  │  + Index    │  traces-apm-*, metrics-apm-*
                  │  Management │  logs-apm-*
                  └──────┬──────┘
                         │
                         ▼
                  ┌─────────────┐
                  │   Kibana    │  APM UI, dashboards, alerts
                  │  APM UI     │  Correlate logs, traces, metrics
                  └─────────────┘
```

### Docker Compose Setup

```yaml
version: "3.9"

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
    environment:
      discovery.type: single-node
      ES_JAVA_OPTS: "-Xms2g -Xmx2g"
      ELASTIC_PASSWORD: "${ELASTIC_PASSWORD}"
      xpack.security.enabled: "true"
    ports:
      - "9200:9200"
    volumes:
      - esdata:/usr/share/elasticsearch/data

  apm-server:
    image: docker.elastic.co/apm/apm-server:8.13.0
    command:
      - --strict.perms=false
    volumes:
      - ./apm-server.yml:/usr/share/apm-server/apm-server.yml:ro
    ports:
      - "8200:8200"
    environment:
      - output.elasticsearch.hosts=["elasticsearch:9200"]
      - output.elasticsearch.username=elastic
      - output.elasticsearch.password=${ELASTIC_PASSWORD}
      - apm-server.kibana.enabled=true
      - apm-server.kibana.host=kibana:5601
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.0
    environment:
      ELASTICSEARCH_HOSTS: "http://elasticsearch:9200"
      ELASTICSEARCH_USERNAME: elastic
      ELASTICSEARCH_PASSWORD: "${ELASTIC_PASSWORD}"
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  payment-service:
    image: myregistry/payment-service:latest
    environment:
      ELASTIC_APM_SERVICE_NAME: payment-service
      ELASTIC_APM_SERVER_URL: http://apm-server:8200
      ELASTIC_APM_ENVIRONMENT: production
      ELASTIC_APM_SERVICE_VERSION: "1.2.3"
      ELASTIC_APM_LOG_LEVEL: info
    depends_on:
      - apm-server

volumes:
  esdata:
```

### Python Elastic APM Agent

```python
# Install
# pip install elastic-apm

from elasticapm.contrib.flask import ElasticAPM
from flask import Flask

app = Flask(__name__)
app.config['ELASTIC_APM'] = {
    'SERVICE_NAME': 'payment-service',
    'SERVER_URL': 'http://apm-server:8200',
    'ENVIRONMENT': 'production',
    'SERVICE_VERSION': '1.2.3',
    'CAPTURE_HEADERS': True,
    'CAPTURE_BODY': 'errors',    # Capture request body on errors only
    'TRANSACTION_SAMPLE_RATE': 1.0,
}
apm = ElasticAPM(app)

@app.route('/payments', methods=['POST'])
def create_payment():
    # APM auto-instruments this endpoint
    # For custom spans:
    with apm.client.capture_span('validate_payment', span_type='custom'):
        validate()

    # Custom context
    apm.client.get_transaction().label(order_id='order-123', currency='USD')
    return {'status': 'ok'}
```

### Java Elastic APM Agent

```bash
# Download
curl -o elastic-apm-agent.jar \
  https://search.maven.org/remotecontent?filepath=co/elastic/apm/elastic-apm-agent/LATEST/elastic-apm-agent-LATEST.jar

# Run with agent
java -javaagent:elastic-apm-agent.jar \
     -Delastic.apm.service_name=payment-service \
     -Delastic.apm.server_url=http://apm-server:8200 \
     -Delastic.apm.environment=production \
     -Delastic.apm.application_packages=com.example \
     -Delastic.apm.log_level=INFO \
     -jar myapp.jar
```

### KQL Queries in Kibana APM

```
// Kibana Query Language for APM data

// Find all traces for a specific user
user.id: "user-12345" AND transaction.type: "request"

// Slow transactions
transaction.duration.us > 2000000 AND service.name: "payment-service"

// Errors in the last hour  
error.exception.type: "SQLException" AND @timestamp >= now-1h

// Traces with specific span type
span.type: "db" AND span.subtype: "postgresql" AND span.duration.us > 100000

// Find traces for a specific endpoint
transaction.name: "POST /api/v1/payments" AND transaction.result: "HTTP 5xx"

// High error rates by service
service.name: * AND error.id: *
```

---

## Continuous Profiling

### What it is and Why it Matters

**Continuous profiling** is the practice of running profilers in production, continuously, at low overhead. Unlike traditional profiling (run locally on synthetic workloads), continuous profiling captures real production behavior.

```
Traditional Profiling vs. Continuous Profiling
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Traditional:
  Engineer suspects perf issue → runs profiler on local machine
  → synthetic workload → different CPU, memory, data than prod
  → 30% overhead → can't run in prod → may miss the actual issue

  Continuous:
  Always on in prod → captures actual user traffic
  → 1-5% overhead → always available
  → "Which commit made checkout 200ms slower?" → instant answer
  → Correlate with traces: "This slow trace → this hot function"
```

### CPU Profiling

CPU profiling samples the call stack at regular intervals (typically 100 Hz). The resulting data shows where the CPU spends its time.

```
CPU Flame Graph interpretation
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Width = CPU time consumed (wider = more CPU used)
  Height = call stack depth
  Color = package/module (visual grouping only)

  processPayment [████████████████████████████████] 80% CPU
    ├── validateAmount [██] 5%
    ├── checkFraud [████████████████████] 50% CPU ← HOT PATH
    │     └── computeMLScore [████████████████████]
    │           └── matrix.Multiply [████████████] ← HOTTEST
    └── saveToDatabase [████████] 25%
```

### Memory Profiling

Memory profiling captures allocations and heap profiles to identify memory leaks and excessive garbage collection.

```python
# Python memory profiling with tracemalloc
import tracemalloc

tracemalloc.start()
# ... run workload ...
snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics('lineno')
for stat in top_stats[:10]:
    print(stat)
```

### Goroutine Profiling (Go)

```go
// Go: expose pprof endpoints
import _ "net/http/pprof"

func main() {
    go func() {
        // pprof endpoint at /debug/pprof/
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    // ... rest of app
}

// Collect profiles:
// CPU: go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
// Memory: go tool pprof http://localhost:6060/debug/pprof/heap
// Goroutines: go tool pprof http://localhost:6060/debug/pprof/goroutine
// Mutex contention: go tool pprof http://localhost:6060/debug/pprof/mutex
// Block profile: go tool pprof http://localhost:6060/debug/pprof/block
```

### Profiling Overhead Considerations

| Profiler Type | Typical CPU Overhead | Memory Overhead | Notes |
|--------------|---------------------|----------------|-------|
| CPU sampling (100 Hz) | 1–3% | Negligible | Safe for always-on production |
| Memory allocation | 3–7% | 2–5% heap | Consider sampling 10% of allocs |
| Mutex profiling | 2–4% | Negligible | Enable only when investigating |
| Block profiling | 5–10% | Low | Enable only when investigating |
| eBPF profiling | < 1% | Minimal | Kernel-level, zero app changes |

---

## Pyroscope

### Architecture

```
Pyroscope Architecture
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Applications (SDK push or pull mode)
  ┌───────────────────────────────────────────────────────────┐
  │  Go SDK  │  Python SDK  │  Java SDK  │  eBPF agent        │
  └─────┬────┴──────┬───────┴──────┬─────┴────────┬──────────┘
        │           │              │               │
        └───────────┼──────────────┼───────────────┘
                    │ HTTP POST profile data
                    ▼
            ┌─────────────┐
            │  Pyroscope  │  Ingestion, storage, query server
            │   Server    │  Compatible with Grafana data source
            └──────┬──────┘
                   │
                   ▼
            ┌─────────────┐
            │   Grafana   │  Flame graph visualization
            │  Pyroscope  │  Compare profiles between time ranges
            │  Data Source│  Correlate with traces/metrics
            └─────────────┘
```

### Docker Deployment

```yaml
version: "3.9"
services:
  pyroscope:
    image: grafana/pyroscope:latest
    ports:
      - "4040:4040"
    volumes:
      - pyroscope_data:/data
    command:
      - server
      - --storage.path=/data
      - --log.level=info

volumes:
  pyroscope_data:
```

### Go Profiling Integration

```go
package main

import (
    "context"
    "log"

    "github.com/grafana/pyroscope-go"
)

func main() {
    profiler, err := pyroscope.Start(pyroscope.Config{
        ApplicationName: "payment-service",
        ServerAddress:   "http://pyroscope:4040",
        Logger:          pyroscope.StandardLogger,
        Tags: map[string]string{
            "env":     "production",
            "version": "1.2.3",
            "region":  "us-east-1",
        },

        // Profile types to collect
        ProfileTypes: []pyroscope.ProfileType{
            pyroscope.ProfileCPU,
            pyroscope.ProfileAllocObjects,
            pyroscope.ProfileAllocSpace,
            pyroscope.ProfileInuseObjects,
            pyroscope.ProfileInuseSpace,
            pyroscope.ProfileGoroutines,
            pyroscope.ProfileMutexCount,
            pyroscope.ProfileMutexDuration,
            pyroscope.ProfileBlockCount,
            pyroscope.ProfileBlockDuration,
        },
    })
    if err != nil {
        log.Fatal(err)
    }
    defer profiler.Stop()

    // Tag code sections for label-based filtering
    // Labels allow you to isolate profiles by code path
    pyroscope.TagWrapper(context.Background(),
        pyroscope.Labels("handler", "ProcessPayment"),
        func(ctx context.Context) {
            processPayment(ctx)
        },
    )
}
```

### Python Profiling Integration

```python
import pyroscope

pyroscope.configure(
    application_name="payment-service",
    server_address="http://pyroscope:4040",
    auth_token="",            # if auth enabled
    tags={
        "env": "production",
        "region": "us-east-1",
    },
    enable_logging=True,
    detect_subprocesses=False,
)

# Tag specific code sections
with pyroscope.tag_wrapper({"handler": "process_payment"}):
    result = process_payment(order_id, amount)
```

### Flame Graph Interpretation Guide

```
Flame Graph Reading Guide
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  What to look for:

  1. WIDE BARS AT TOP = hot functions (consuming most CPU time)
     → Optimization target: these give the biggest bang for effort

  2. WIDE PLATEAU IN MIDDLE = all CPU consumed in one subtree
     → Check: is this expected? Deep call chains often indicate
        blocking I/O or inefficient algorithms

  3. TALL, NARROW STACKS = deep recursion or proxy chains
     → Usually not a problem unless the base is wide

  4. FLAT TOP = CPU time evenly distributed, no clear hotspot
     → May indicate the load is inherently distributed; good!
     → May indicate profiling isn't working; check config

  Comparing profiles (diff flame graphs):
  → RED  = function takes MORE time in new version (regression)
  → BLUE = function takes LESS time in new version (improvement)
  → GRAY = unchanged

  Pro tip: Use Pyroscope's "Compare" view to compare:
  → Before deploy vs. after deploy
  → Peak hours vs. off-peak
  → Canary vs. stable release
```

---

## eBPF-based Profiling

### What eBPF is and Why it's Powerful for Profiling

**eBPF (extended Berkeley Packet Filter)** is a kernel technology that allows programs to run sandboxed code inside the Linux kernel without modifying kernel source code or loading kernel modules. For profiling:

```
eBPF Profiling vs. Traditional Profiling
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Traditional (SDK-based):
  App → SDK calls → Profile data → Push to server
  Requires: SDK integration, code changes, per-language support
  Overhead: 1–5% CPU per process

  eBPF-based:
  Kernel → eBPF program runs on scheduler events → Profile data
  Requires: ZERO application changes
  Languages: ALL (Go, Java, Python, C++, Rust, Ruby, PHP, Node.js)
  Overhead: < 1% system-wide

  Kernel events tapped:
  - CPU scheduler events (perf_event)
  - Function entry/exit (uprobes/kprobes)
  - Memory allocation events
  - I/O events
```

### Parca: Architecture and Deployment

**Parca** is an open-source continuous profiling tool using eBPF.

```bash
# Deploy Parca Agent (DaemonSet — runs on every node)
kubectl apply -f https://github.com/parca-dev/parca-agent/releases/latest/download/kubernetes-manifest.yaml

# Deploy Parca Server
kubectl apply -f https://github.com/parca-dev/parca/releases/latest/download/kubernetes-manifest.yaml
```

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: parca-agent
  namespace: parca
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: parca-agent
  template:
    spec:
      hostPID: true       # Required for eBPF
      hostNetwork: true   # Required for eBPF
      containers:
        - name: parca-agent
          image: ghcr.io/parca-dev/parca-agent:latest
          args:
            - --node=$(NODE_NAME)
            - --remote-store-address=parca.parca.svc.cluster.local:7070
            - --remote-store-insecure
          securityContext:
            privileged: true    # Required for eBPF
          volumeMounts:
            - mountPath: /sys
              name: sys
              readOnly: true
            - mountPath: /proc
              name: proc
              readOnly: true
      volumes:
        - name: sys
          hostPath:
            path: /sys
        - name: proc
          hostPath:
            path: /proc
```

### Grafana Pyroscope eBPF Agent

```yaml
# Grafana Alloy (formerly Grafana Agent) — eBPF profiling config
pyroscope.ebpf "scrape_all" {
  targets = [
    {__address__ = "localhost"},
  ]
  forward_to = [pyroscope.write.endpoint.receiver]
  demangle = "none"
}

pyroscope.write "endpoint" {
  endpoint {
    url = "http://pyroscope:4040"
  }
}
```

### Benefits vs Traditional Profiling

| Benefit | eBPF | Traditional SDK |
|---------|------|----------------|
| Zero code changes | ✅ | ❌ Requires SDK integration |
| All languages | ✅ Native, JVM, interpreted | ❌ Per-language SDK |
| CPU overhead | < 1% | 1–5% per process |
| Kernel visibility | ✅ Syscalls, I/O | ❌ User-space only |
| Production safe | ✅ Sandboxed by kernel | ⚠️ Higher overhead |
| Detailed app context | ❌ No business logic labels | ✅ Full control over labels |
| JVM profiling | ⚠️ Sees JVM, not JIT code | ✅ JVM-aware frame walking |

---

## Performance Baselines and Anomaly Detection

### How to Establish Baselines

A **baseline** is a statistical model of "normal" behavior for a metric. Anomaly detection compares current behavior to the baseline to identify deviations.

```
Baseline Establishment Process
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Step 1: Collect data (minimum 2–4 weeks)
    → Capture weekday/weekend patterns
    → Capture time-of-day patterns
    → Capture seasonal spikes (end of month, Black Friday)

  Step 2: Model the distribution
    → P50, P75, P90, P99 for latency metrics
    → Mean ± N standard deviations for volume metrics
    → Hour-of-week bucketing for periodic patterns

  Step 3: Set threshold as multiple of baseline
    → Alert at 3× the typical value for latency
    → Alert at 5σ deviation for error rate
    → Never use absolute thresholds without baseline context
```

### Static vs Dynamic Thresholds

| Threshold Type | Definition | Pros | Cons | Best For |
|----------------|-----------|------|------|----------|
| **Static** | Fixed value (e.g., latency > 500ms) | Simple, predictable | Misses seasonal variation; too many false positives at peak | SLO-bound metrics with guaranteed maximums |
| **Dynamic** | Multiple of baseline (e.g., 3× typical p99) | Adapts to normal variation | Requires training data; can miss slow-growing issues | Traffic-correlated metrics like latency, throughput |
| **Seasonal** | Compares to same hour last week | Handles day-of-week patterns | Requires > 1 week of data | E-commerce with weekly patterns |

### ML-based Anomaly Detection

Major APM platforms provide ML-based anomaly detection:

| Platform | Technology | Configuration |
|----------|-----------|--------------|
| **Datadog Watchdog** | Automated, no config needed | Shows in "Watchdog Insights" sidebar automatically |
| **New Relic AI** | Automatic baseline per signal | Applied Intelligence → Configure Anomaly Detection |
| **Dynatrace Davis** | Fully automated | No configuration; Davis observes all metrics |
| **Elastic ML** | Manual ML jobs | Machine Learning → Anomaly Detection → Create Job |
| **Grafana MLflow** | Requires setup | Grafana ML plugin → Outlier detection |

---

## APM Comparison Table

| Feature | Datadog | New Relic | Dynatrace | Elastic APM | Grafana (OSS) |
|---------|---------|-----------|-----------|-------------|----------------|
| **Pricing model** | Per host + custom metrics | Per data ingest (GB) | Per host (DEM units) | Per GB ingest | Free (OSS) |
| **Java auto-instrumentation** | ✅ | ✅ | ✅ (OneAgent, all) | ✅ | ✅ (via OTel) |
| **Python auto-instrumentation** | ✅ | ✅ | ✅ | ✅ | ✅ (via OTel) |
| **Go auto-instrumentation** | Partial | Partial | ✅ (native) | ✅ | Via OTel SDK |
| **Node.js auto-instrumentation** | ✅ | ✅ | ✅ | ✅ | ✅ (via OTel) |
| **Distributed tracing** | ✅ Sampled | ✅ Sampled | ✅ 100% (PurePath) | ✅ Sampled | ✅ Jaeger/Tempo |
| **Continuous profiling** | ✅ (extra cost) | ✅ (Pixie) | ✅ (extra cost) | ❌ | ✅ Pyroscope |
| **Real User Monitoring** | ✅ | ✅ | ✅ | ✅ | Limited |
| **Synthetic monitoring** | ✅ | ✅ | ✅ | ✅ | ✅ k6 |
| **Log correlation** | ✅ | ✅ | ✅ | ✅ (native) | ✅ Loki+Tempo |
| **Infrastructure monitoring** | ✅ Extensive | ✅ | ✅ (Smartscape) | ✅ | ✅ (separate) |
| **AI/ML anomaly detection** | ✅ Watchdog | ✅ AI | ✅ Davis (best) | Partial ML jobs | ❌ |
| **Kubernetes native** | ✅ | ✅ | ✅ Operator | ✅ | ✅ |
| **Open source** | ❌ | ❌ | ❌ | ✅ (Apache 2.0) | ✅ |
| **Self-hosted option** | ❌ | ❌ | ❌ | ✅ | ✅ |
| **OTLP native support** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Database monitoring** | ✅ DBM addon | ✅ | ✅ | ✅ | Basic |
| **Network monitoring** | ✅ NPM addon | ✅ | ✅ | Limited | Via Beyla/eBPF |

---

## Cost Considerations

Understanding APM cost drivers prevents bill shock and helps you optimize spend.

| Cost Driver | Datadog | New Relic | Dynatrace | Elastic APM | Grafana Cloud |
|-------------|---------|-----------|-----------|-------------|---------------|
| **Infrastructure** | Per host/container | Included | Per host (Full Stack) | Elasticsearch cluster | Per active series |
| **Custom metrics** | $0.05/metric/month | Per GB | Per DEM unit | Per GB ingest | Per active series |
| **Traces (APM)** | Per host + retained spans | Per GB ingest | Included with host | Per GB ingest | Per trace |
| **Logs** | Per GB ingest + retention | Per GB ingest | Per GB ingest | Per GB ingest + storage | Per GB ingest |
| **Profiling** | Per profiled host | Included | Per host | Not included | Pyroscope free |
| **RUM** | Per 10K sessions | Per GB ingest | Per DEM unit | Per GB ingest | Per session |
| **Synthetic checks** | Per 10K API tests | Per synthetic check | Per DEM unit | Via k6 integration | Per check |
| **Retention** | 15d default; paid extension | 8d default | 35d default | You control | You control |

**Cost optimization strategies:**

```
APM Cost Optimization
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  1. SAMPLING
     Sample 100% of error traces, 10% of success traces.
     Most vendors charge by ingested volume.
     Tail-based sampling (via OTel Collector) = same visibility, 90% cost reduction.

  2. CUSTOM METRICS
     Review custom metric count monthly.
     Use OTel histogram instead of 4 separate percentile metrics.
     Drop metrics from pre-prod environments.

  3. LOG CORRELATION ONLY (no log storage in APM)
     Keep logs in cheaper storage (S3 + Athena).
     Only send trace/error context to APM vendor.

  4. TIERED RETENTION
     Hot data (7d): full resolution, indexed
     Warm data (30d): downsampled metrics, sampled traces
     Cold data (1y): aggregates only, no raw traces

  5. ENVIRONMENT SEPARATION
     Use full APM only in production.
     Use Prometheus + Grafana OSS in dev/staging.
     Saves 60–80% of APM spend.
```

---

## Next Steps

1. **Choose your APM strategy** — Use the comparison table to select a commercial APM (Datadog, New Relic, Dynatrace) or invest in the OSS stack (OTel + Grafana Tempo + Pyroscope + Elastic APM).
2. **Start with auto-instrumentation** — Deploy the Java/Python/Node.js agent for your highest-traffic service first. Capture the baseline before adding custom spans.
3. **Set up continuous profiling** — Deploy Pyroscope or enable continuous profiling in your commercial APM. This is one of the highest ROI observability investments.
4. **Implement distributed tracing across all services** — Ensure every service propagates trace context headers. Use the OTel Collector as the central point of collection.
5. **Build service dependency maps** — Validate that your APM service map matches your architecture diagram. Differences reveal undocumented dependencies.
6. **Configure error grouping** — Set up error fingerprinting to group similar exceptions. Track error volumes across deploys to catch regressions.
7. **Implement RUM** — Add Real User Monitoring to correlate backend performance with actual user-perceived performance.

**Related guides in this series:**
- [04-OPENTELEMETRY.md](04-OPENTELEMETRY.md) — OTel SDK and Collector for vendor-neutral instrumentation
- [03-DISTRIBUTED-TRACING.md](03-DISTRIBUTED-TRACING.md) — Jaeger and Zipkin for OSS tracing
- [07-SLIS-SLOS-SLAS.md](07-SLIS-SLOS-SLAS.md) — Using APM data to define and track SLOs

---

## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2025-01-01 | Platform Engineering | Initial document |
| 1.1.0 | 2025-01-20 | Platform Engineering | Added Dynatrace DQL and Davis AI sections |
| 1.2.0 | 2025-02-15 | Platform Engineering | Added Pyroscope and eBPF profiling sections |
| 1.3.0 | 2025-03-10 | Platform Engineering | Expanded Elastic APM with full Docker Compose |
| 1.4.0 | 2025-04-05 | Platform Engineering | Added cost optimization section and comparison table |
| 1.5.0 | 2025-06-01 | Platform Engineering | Added Parca deployment and Grafana Pyroscope eBPF config |
