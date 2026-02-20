# Metrics

## Table of Contents

1. [What Are Metrics](#what-are-metrics)
2. [Metric Types](#metric-types)
   - [Counter](#counter)
   - [Gauge](#gauge)
   - [Histogram](#histogram)
   - [Summary](#summary)
3. [Prometheus Architecture](#prometheus-architecture)
4. [Prometheus Installation](#prometheus-installation)
5. [Prometheus Configuration](#prometheus-configuration)
6. [Instrumenting Applications](#instrumenting-applications)
   - [Go](#go-instrumentation)
   - [Python](#python-instrumentation)
   - [Java](#java-instrumentation)
7. [PromQL](#promql)
8. [Recording Rules](#recording-rules)
9. [Grafana Dashboards](#grafana-dashboards)
10. [Prometheus Federation](#prometheus-federation)
11. [Remote Storage Backends](#remote-storage-backends)
12. [Metric Naming Best Practices](#metric-naming-best-practices)
13. [Next Steps](#next-steps)

---

## What Are Metrics

Metrics are **numeric measurements sampled over time**. They are the most efficient and cheapest form of telemetry to collect, store, and query at scale. A single metric time series consists of:

- A **metric name**: human-readable identifier (e.g., `http_requests_total`)
- A **timestamp**: when the sample was taken (millisecond Unix epoch in Prometheus)
- A **value**: a 64-bit floating-point number
- A set of **labels** (also called dimensions or tags): key-value pairs that allow filtering and aggregation

```
 Metric name          Labels                                        Value  Timestamp
     │                  │                                             │       │
     ▼                  ▼                                             ▼       ▼
http_requests_total{method="POST", path="/api/orders", status="200"} 4827   1687200000000
```

### Time-Series Concepts

```
  Value
   │
   │                    ●
  800│              ●       ●
   │           ●               ●
  400│      ●                       ●
   │    ●                               ●
    └─────────────────────────────────────► Time
     t0   t1   t2   t3   t4   t5   t6   t7

  Each (name + label set) → one time series
  Multiple scrapes         → data points on that series
  PromQL                   → queries across time series
```

### Labels and Cardinality

Labels define the **cardinality** of a metric. Every unique combination of label values creates a new time series. This is powerful for filtering, but cardinality must be controlled carefully.

| Label Usage | Good | Bad |
|---|---|---|
| `status="200"` | ✅ Low cardinality (finite HTTP status codes) | — |
| `method="GET"` | ✅ Low cardinality (handful of HTTP verbs) | — |
| `user_id="usr_abc123"` | — | ❌ High cardinality — millions of series |
| `url="/api/orders/1234"` | — | ❌ Per-request URLs explode series count |

---

## Metric Types

Prometheus defines four core metric types. Understanding which type to use for each measurement is foundational to good instrumentation.

### Counter

A **counter** is a cumulative metric that only ever increases (or resets to zero on process restart). Counters represent totals: total requests, total errors, total bytes sent.

**Never use a counter for values that can decrease** (use a Gauge for those).

**When to use:** Total number of events (requests, errors, bytes, tasks completed)

```go
// Go — using prometheus/client_golang
package main

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var httpRequestsTotal = promauto.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total number of HTTP requests processed.",
    },
    []string{"method", "path", "status"},
)

// In your HTTP handler:
func handler(w http.ResponseWriter, r *http.Request) {
    // ... handle request ...
    httpRequestsTotal.WithLabelValues("GET", "/api/orders", "200").Inc()
}
```

```python
# Python — using prometheus_client
from prometheus_client import Counter

http_requests_total = Counter(
    'http_requests_total',
    'Total number of HTTP requests processed.',
    ['method', 'path', 'status']
)

# In your route handler:
def handle_request(method, path, status):
    http_requests_total.labels(method=method, path=path, status=status).inc()
```

### Gauge

A **gauge** is a metric that represents a single numerical value that can arbitrarily go up and down. Gauges represent snapshots of a current state.

**When to use:** Current values like queue depth, active connections, temperature, memory usage, in-flight requests

```go
// Go — Gauge example
var activeConnections = promauto.NewGauge(prometheus.GaugeOpts{
    Name: "active_connections",
    Help: "Number of currently active connections.",
})

// When a connection opens:
activeConnections.Inc()

// When a connection closes:
activeConnections.Dec()

// Or set an absolute value:
activeConnections.Set(42)
```

```python
# Python — Gauge example
from prometheus_client import Gauge

active_connections = Gauge(
    'active_connections',
    'Number of currently active connections.'
)

# Increment / decrement
active_connections.inc()
active_connections.dec()

# Use as a context manager for in-flight tracking
with active_connections.track_inprogress():
    process_request()
```

### Histogram

A **histogram** samples observations (usually request durations or payload sizes) and counts them in configurable buckets. It also provides the total sum and total count of observations. This allows computing **quantiles** (percentiles) using `histogram_quantile()` in PromQL.

A histogram exposes three time series per label set:
- `<name>_bucket{le="<bound>"}` — cumulative count up to each bucket boundary
- `<name>_sum` — sum of all observed values
- `<name>_count` — total number of observations

**When to use:** Request durations, response sizes, batch processing times — anything you want to compute quantiles for

```go
// Go — Histogram example
var httpDuration = promauto.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Help:    "HTTP request latency distribution.",
        Buckets: prometheus.DefBuckets, // .005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10
    },
    []string{"method", "path"},
)

func handler(w http.ResponseWriter, r *http.Request) {
    start := time.Now()
    // ... handle request ...
    elapsed := time.Since(start).Seconds()
    httpDuration.WithLabelValues(r.Method, r.URL.Path).Observe(elapsed)
}
```

```python
# Python — Histogram example
from prometheus_client import Histogram

http_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency distribution.',
    ['method', 'path'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

# Use as a decorator or context manager
@http_duration.labels(method='GET', path='/api/orders').time()
def handle_get_orders():
    pass
```

### Summary

A **summary** also samples observations, but it calculates configurable quantiles **on the client side** (over a sliding time window), then exposes them directly. Unlike histograms, summaries cannot be aggregated across instances.

| Dimension | Histogram | Summary |
|---|---|---|
| **Quantile calculation** | Server-side (PromQL) | Client-side |
| **Aggregatable across instances** | ✅ Yes | ❌ No |
| **Requires predefined buckets** | ✅ Yes | ❌ No (predefined quantiles) |
| **Accurate quantiles** | Approximate (depends on buckets) | Accurate (for that instance) |
| **Recommended** | ✅ Prefer histogram in most cases | Only when you need exact quantiles on a single instance |

```go
// Go — Summary example
var requestDurationSummary = promauto.NewSummaryVec(
    prometheus.SummaryOpts{
        Name:       "rpc_duration_seconds",
        Help:       "RPC duration summary.",
        Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
    },
    []string{"method"},
)
```

```python
# Python — Summary example
from prometheus_client import Summary

rpc_duration = Summary(
    'rpc_duration_seconds',
    'RPC duration summary.',
    ['method']
)

with rpc_duration.labels(method='GetOrder').time():
    call_rpc()
```

---

## Prometheus Architecture

```
                    ┌────────────────────────────────────────────────────────┐
                    │                    Prometheus Server                   │
                    │                                                        │
  ┌───────────┐     │  ┌────────────┐    ┌──────────┐    ┌───────────────┐  │
  │ App A     │◄────┼──│            │    │          │    │               │  │
  │ /metrics  │     │  │  Scrape    │───►│   TSDB   │───►│    PromQL     │  │
  └───────────┘     │  │  Engine   │    │  Storage │    │    Engine     │  │
                    │  │           │    │          │    │               │  │
  ┌───────────┐     │  │  (pull    │    └──────────┘    └───────┬───────┘  │
  │ App B     │◄────┼──│   model)  │                            │           │
  │ /metrics  │     │  │           │    ┌──────────┐            │           │
  └───────────┘     │  └─────┬─────┘    │  Rules   │◄───────────┤           │
                    │        │          │  Engine  │            │           │
  ┌───────────┐     │        │          └─────┬────┘    ┌───────▼───────┐   │
  │ Node      │◄────┘        │                │         │  HTTP API     │   │
  │ Exporter  │              │                ▼         └───────────────┘   │
  └───────────┘              │     ┌──────────────────┐         │           │
                             │     │  Alertmanager    │         │           │
  ┌───────────┐              │     │                  │   ┌─────▼──────┐    │
  │Pushgateway│◄─────────────┘     │  Routes/Silence  │   │  Grafana   │    │
  │(batch jobs│                    │  PagerDuty/Slack  │   │  Dashboards│    │
  └───────────┘                    └──────────────────┘   └────────────┘    │
                                                                            │
  ┌────────────────────────────────────────────────────────────────────────┘
  │  Service Discovery: Kubernetes, Consul, EC2, DNS, file-based
```

### Components

| Component | Role |
|---|---|
| **Prometheus Server** | Core scraping, storage (TSDB), and query engine |
| **TSDB** | Custom time-series database; data stored in blocks with local WAL |
| **PromQL Engine** | Functional query language for metrics analysis and alerting |
| **Alertmanager** | Deduplication, grouping, inhibition, and routing of alerts |
| **Pushgateway** | Accepts metrics pushed by short-lived batch jobs |
| **Exporters** | Third-party metric adapters (node_exporter, mysqld_exporter, etc.) |
| **Service Discovery** | Dynamic target discovery (Kubernetes, Consul, EC2, DNS) |

---

## Prometheus Installation

### Docker Compose (Local Development)

```yaml
# docker-compose.yml
version: "3.8"
services:
  prometheus:
    image: prom/prometheus:v2.48.0
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'          # Enables /-/reload endpoint
      - '--web.enable-admin-api'
    ports:
      - "9090:9090"

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    network_mode: host
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/)'

volumes:
  prometheus_data:
```

```bash
# Start
docker compose up -d

# Reload config without restarting
curl -X POST http://localhost:9090/-/reload

# Verify targets
open http://localhost:9090/targets
```

### Kubernetes (kube-prometheus-stack Helm Chart)

```bash
# Add the prometheus-community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack (includes Prometheus, Grafana, Alertmanager, node-exporter)
helm install kube-prom-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set grafana.adminPassword=supersecret

# Check the pods
kubectl -n monitoring get pods

# Port-forward Prometheus UI
kubectl -n monitoring port-forward svc/kube-prom-stack-kube-prome-prometheus 9090:9090

# Port-forward Grafana UI
kubectl -n monitoring port-forward svc/kube-prom-stack-grafana 3000:80
```

---

## Prometheus Configuration

```yaml
# prometheus.yml — complete example with all major sections

# Global settings applied to all scrape jobs unless overridden
global:
  scrape_interval: 15s       # How often to scrape targets
  evaluation_interval: 15s   # How often to evaluate alerting/recording rules
  scrape_timeout: 10s        # Timeout per scrape request

  # Labels added to all time series and alerts produced by this Prometheus
  external_labels:
    cluster: production
    region: us-east-1

# Alertmanager connection configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
      # Optional: use Kubernetes service discovery for alertmanager
      # kubernetes_sd_configs:
      #   - role: endpoints
      #     namespaces:
      #       names: [monitoring]

# Files containing recording and alerting rules
rule_files:
  - /etc/prometheus/rules/*.yml
  - /etc/prometheus/alerts/*.yml

# Scrape configuration — defines what to scrape and how
scrape_configs:

  # Prometheus self-monitoring
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  # Node exporter (host-level metrics)
  - job_name: node-exporter
    static_configs:
      - targets: ['node-exporter:9100']
    # Relabeling: add an 'env' label based on the target
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance

  # Application service — static targets
  - job_name: order-service
    scrape_interval: 10s     # Override global for this job
    metrics_path: /metrics   # Default; customize if your app uses a different path
    scheme: http             # http or https
    static_configs:
      - targets: ['order-service:8080']
        labels:
          team: checkout
          env: production

  # Kubernetes pod auto-discovery (requires Prometheus inside a cluster)
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape pods with the annotation prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      # Use the path from the annotation prometheus.io/path (default /metrics)
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      # Use the port from the annotation prometheus.io/port
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      # Expose pod namespace and name as labels
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

  # Kubernetes node auto-discovery
  - job_name: kubernetes-nodes
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
```

---

## Instrumenting Applications

### Go Instrumentation

```go
// main.go — complete working Go application with Prometheus instrumentation
package main

import (
    "fmt"
    "math/rand"
    "net/http"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    // Counter: total requests by method, path, and status
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests.",
        },
        []string{"method", "path", "status"},
    )

    // Gauge: current number of active requests
    httpRequestsInFlight = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "http_requests_in_flight",
            Help: "Current number of HTTP requests being processed.",
        },
    )

    // Histogram: request duration distribution
    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds.",
            Buckets: []float64{0.001, 0.005, 0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5},
        },
        []string{"method", "path"},
    )

    // Summary: response size distribution
    httpResponseSize = promauto.NewSummaryVec(
        prometheus.SummaryOpts{
            Name: "http_response_size_bytes",
            Help: "HTTP response size in bytes.",
            Objectives: map[float64]float64{
                0.5:  0.05,
                0.9:  0.01,
                0.99: 0.001,
            },
        },
        []string{"method", "path"},
    )
)

func instrumentedHandler(path string, handler http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        httpRequestsInFlight.Inc()
        defer httpRequestsInFlight.Dec()

        // Use a response writer wrapper to capture the status code
        rw := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
        handler(rw, r)

        elapsed := time.Since(start).Seconds()
        status := fmt.Sprintf("%d", rw.statusCode)

        httpRequestsTotal.WithLabelValues(r.Method, path, status).Inc()
        httpRequestDuration.WithLabelValues(r.Method, path).Observe(elapsed)
        httpResponseSize.WithLabelValues(r.Method, path).Observe(float64(rw.bytesWritten))
    }
}

type responseWriter struct {
    http.ResponseWriter
    statusCode   int
    bytesWritten int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
    n, err := rw.ResponseWriter.Write(b)
    rw.bytesWritten += n
    return n, err
}

func ordersHandler(w http.ResponseWriter, r *http.Request) {
    // Simulate variable latency
    time.Sleep(time.Duration(rand.Intn(200)) * time.Millisecond)
    fmt.Fprintln(w, `{"orders": []}`)
}

func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.HandleFunc("/api/orders", instrumentedHandler("/api/orders", ordersHandler))

    fmt.Println("Listening on :8080")
    http.ListenAndServe(":8080", nil)
}
```

```bash
# Install dependencies
go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promauto
go get github.com/prometheus/client_golang/prometheus/promhttp

# Run and test
go run main.go &
curl http://localhost:8080/api/orders
curl http://localhost:8080/metrics | grep http_requests
```

### Python Instrumentation

```python
# app.py — Flask application with Prometheus instrumentation
from flask import Flask, jsonify
from prometheus_client import (
    Counter, Gauge, Histogram, Summary,
    generate_latest, CONTENT_TYPE_LATEST
)
import time
import random
import functools

app = Flask(__name__)

# Metrics definitions
http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests.',
    ['method', 'endpoint', 'status']
)

http_requests_in_flight = Gauge(
    'http_requests_in_flight',
    'Number of HTTP requests currently being processed.'
)

http_request_duration_seconds = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration.',
    ['method', 'endpoint'],
    buckets=[0.001, 0.005, 0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

http_response_size_bytes = Summary(
    'http_response_size_bytes',
    'HTTP response size.',
    ['method', 'endpoint']
)


def track_metrics(endpoint):
    """Decorator to add Prometheus instrumentation to a route handler."""
    def decorator(f):
        @functools.wraps(f)
        def wrapper(*args, **kwargs):
            method = 'GET'  # simplified; use flask.request.method in practice
            start = time.time()
            http_requests_in_flight.inc()
            try:
                response = f(*args, **kwargs)
                status = str(response.status_code)
            except Exception:
                status = '500'
                raise
            finally:
                duration = time.time() - start
                http_requests_in_flight.dec()
                http_requests_total.labels(
                    method=method, endpoint=endpoint, status=status
                ).inc()
                http_request_duration_seconds.labels(
                    method=method, endpoint=endpoint
                ).observe(duration)
            return response
        return wrapper
    return decorator


@app.route('/api/orders')
@track_metrics('/api/orders')
def get_orders():
    time.sleep(random.uniform(0.01, 0.2))  # simulate latency
    return jsonify(orders=[])


@app.route('/metrics')
def metrics():
    return generate_latest(), 200, {'Content-Type': CONTENT_TYPE_LATEST}


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

```bash
pip install flask prometheus-client
python app.py &
curl http://localhost:8080/api/orders
curl http://localhost:8080/metrics
```

### Java Instrumentation

```xml
<!-- pom.xml — add Micrometer + Prometheus registry -->
<dependencies>
  <dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <version>1.12.0</version>
  </dependency>
  <!-- Spring Boot Actuator exposes /actuator/prometheus automatically -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
</dependencies>
```

```java
// OrderController.java — Spring Boot with Micrometer instrumentation
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final Counter ordersCreated;
    private final Timer orderProcessingTime;
    private final MeterRegistry registry;

    public OrderController(MeterRegistry registry) {
        this.registry = registry;

        // Counter: total orders created
        this.ordersCreated = Counter.builder("orders_created_total")
            .description("Total number of orders created.")
            .tag("service", "order-service")
            .register(registry);

        // Timer (equivalent to Prometheus Histogram)
        this.orderProcessingTime = Timer.builder("order_processing_duration_seconds")
            .description("Time to process an order.")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }

    @PostMapping
    public Order createOrder(@RequestBody OrderRequest req) {
        return orderProcessingTime.record(() -> {
            Order order = processOrder(req);
            ordersCreated.increment();
            return order;
        });
    }
}
```

```yaml
# application.yml — expose Prometheus endpoint
management:
  endpoints:
    web:
      exposure:
        include: prometheus, health, info
  metrics:
    export:
      prometheus:
        enabled: true
```

---

## PromQL

PromQL (Prometheus Query Language) is a functional language for selecting and aggregating time-series data. Below are 15 essential queries with explanations.

```promql
# 1. rate() — per-second rate of increase for a counter over the last 5 minutes
#    Use rate() for counters in dashboards and alerts. Never use counters directly.
rate(http_requests_total[5m])


# 2. increase() — absolute increase in a counter over a time window
#    Useful when you want total count in a period (e.g., "errors in last hour")
increase(http_requests_total{status=~"5.."}[1h])


# 3. histogram_quantile() — calculate a quantile from histogram buckets
#    p99 latency across all services, aggregated by service label
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)


# 4. by() aggregation — keep only specified labels after aggregation
#    Total request rate per service (drop all other labels)
sum(rate(http_requests_total[5m])) by (service)


# 5. without() aggregation — drop specified labels, keep everything else
#    Same as above but keeps all labels except 'instance' and 'pod'
sum(rate(http_requests_total[5m])) without (instance, pod)


# 6. topk() — top N time series by value
#    Find the 5 services with the highest error rate right now
topk(5, sum(rate(http_requests_total{status=~"5.."}[5m])) by (service))


# 7. bottomk() — bottom N time series by value
#    Find the 3 least active services
bottomk(3, sum(rate(http_requests_total[5m])) by (service))


# 8. sum() with by() — aggregate across multiple label combinations
#    Total request rate grouped by both service and status class
sum(rate(http_requests_total[5m])) by (service, status)


# 9. avg() — average across all matching time series
#    Average CPU utilization across all nodes in the cluster
avg(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance)


# 10. max_over_time() — maximum value in a range vector
#     Peak memory usage per pod over the last hour (useful for right-sizing)
max_over_time(container_memory_working_set_bytes[1h])


# 11. predict_linear() — linear prediction of a gauge's value in the future
#     Predict disk usage in 4 hours based on the last 1 hour trend
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 4 * 3600) < 0


# 12. absent() — returns 1 if no series match; used to alert on missing metrics
#     Fire an alert if the order-service stops exporting metrics entirely
absent(up{job="order-service"} == 1)


# 13. delta() — calculates the difference between first and last value in a range
#     Change in queue depth over the last 5 minutes (for gauges)
delta(queue_depth[5m])


# 14. irate() — instant rate using last two data points (more responsive than rate)
#     Use irate() for short-duration spikes in dashboards; rate() for alerts
irate(http_requests_total[5m])


# 15. label_replace() — dynamically manipulate label values
#     Extract the service name from a Kubernetes pod label
label_replace(
  up{job="kubernetes-pods"},
  "service",           -- destination label
  "$1",                -- replacement value (capture group)
  "pod",               -- source label
  "(.+)-[a-z0-9]+-[a-z0-9]+"  -- regex with capture group
)
```

---

## Recording Rules

Recording rules pre-compute expensive PromQL expressions and store their result as a new metric. This speeds up dashboards and avoids repeated heavy computations.

```yaml
# rules/recording-rules.yml
groups:
  - name: request_rates
    interval: 30s   # Evaluate every 30 seconds (overrides global evaluation_interval)
    rules:

      # Pre-compute per-service request rate (used in many dashboards)
      - record: job:http_requests_total:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)

      # Pre-compute p99 latency per service
      - record: job:http_request_duration_seconds:p99
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
          )

      # Pre-compute error ratio per service (used in SLO burn rate alerts)
      - record: job:http_requests_error_ratio:rate5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
            /
          sum(rate(http_requests_total[5m])) by (job)
```

---

## Grafana Dashboards

### Connecting to Prometheus (Step-by-Step)

1. Navigate to **Grafana → Administration → Data sources**
2. Click **Add data source** → select **Prometheus**
3. Set **URL** to `http://prometheus:9090` (or `http://localhost:9090` for local)
4. Set **Scrape interval** to match your Prometheus `scrape_interval` (e.g., `15s`)
5. Click **Save & Test** — you should see "Data source is working"

### JSON Panel Example

```json
{
  "type": "timeseries",
  "title": "HTTP Request Rate by Service",
  "datasource": "Prometheus",
  "targets": [
    {
      "expr": "sum(rate(http_requests_total[5m])) by (service)",
      "legendFormat": "{{service}}",
      "refId": "A"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "reqps",
      "color": { "mode": "palette-classic" },
      "custom": {
        "lineWidth": 2,
        "fillOpacity": 10
      }
    }
  },
  "options": {
    "legend": { "displayMode": "table", "placement": "bottom" }
  }
}
```

### Dashboard Variables Example

Variables make dashboards interactive — they become dropdown filters.

```
# In Grafana Dashboard Settings → Variables → Add variable:

Name:        service
Label:       Service
Type:        Query
Data source: Prometheus
Query:       label_values(http_requests_total, service)
Refresh:     On time range change
Sort:        Alphabetical (asc)
```

Then use the variable in panel queries:

```promql
# Use $service variable — user selects from dropdown
sum(rate(http_requests_total{service=~"$service"}[5m])) by (service, status)
```

---

## Prometheus Federation

Federation allows one Prometheus server to scrape metrics from another. This is commonly used to aggregate metrics from multiple cluster-level Prometheus servers into a global Prometheus.

```yaml
# global-prometheus.yml — federation scrape config
scrape_configs:
  - job_name: federate
    scrape_interval: 15s
    honor_labels: true   # Preserve original job/instance labels from the source

    metrics_path: /federate

    params:
      # Only pull recording rule results and job-level aggregations, not raw metrics
      match[]:
        - '{__name__=~"job:.*"}'         # All recording rules with job: prefix
        - '{__name__=~"cluster:.*"}'     # Cluster-level aggregations
        - 'up'                            # Target health

    static_configs:
      - targets:
          - cluster1-prometheus:9090
          - cluster2-prometheus:9090
          - cluster3-prometheus:9090
```

---

## Remote Storage Backends

For long-term storage beyond Prometheus's local retention (default 15 days), use a remote storage backend.

| Backend | Protocol | Best For | Notes |
|---|---|---|---|
| **Thanos** | Prometheus remote read/write | Multi-cluster federation, long-term retention in object storage (S3/GCS) | Sidecar or Receive mode; queries across multiple Prometheus instances |
| **Cortex** | Prometheus remote write | Multi-tenant hosted Prometheus; horizontally scalable writes | CNCF Graduated; basis for Grafana Cloud metrics |
| **Grafana Mimir** | Prometheus remote write | Scalable drop-in replacement for Cortex; Grafana's commercial offering | High write throughput; compacted blocks |
| **VictoriaMetrics** | Prometheus remote write | High-performance, low-cost TSDB; single-node or cluster mode | Drop-in Prometheus replacement with extended MetricsQL |
| **InfluxDB** | Prometheus remote write | Metrics + events; also has its own Flux query language | Popular in IoT and time-series analytics |
| **AWS Managed Prometheus (AMP)** | Prometheus remote write | AWS-native, fully managed Prometheus | No infrastructure to manage; integrates with IAM |
| **Google Cloud Managed Service** | Prometheus remote write | GCP-native managed Prometheus | Monarch-backed; tight GKE integration |

```yaml
# prometheus.yml — configure remote write to Thanos Receiver
remote_write:
  - url: "http://thanos-receive:19291/api/v1/receive"
    remote_timeout: 30s
    queue_config:
      capacity: 10000
      max_shards: 30
      max_samples_per_send: 1000
      batch_send_deadline: 5s
```

---

## Metric Naming Best Practices

| Rule | Good Example | Bad Example |
|---|---|---|
| Use `snake_case` | `http_requests_total` | `httpRequestsTotal`, `HTTP-requests-total` |
| Include unit in name | `request_duration_seconds` | `request_duration`, `request_latency_ms` |
| Counters end in `_total` | `errors_total` | `errors`, `error_count` |
| Use base units (seconds, bytes) | `memory_bytes`, `latency_seconds` | `memory_megabytes`, `latency_milliseconds` |
| Prefix with application/library | `myapp_http_requests_total` | `requests_total` (too generic) |
| Don't include label names in metric name | `requests_total` with label `method` | `get_requests_total`, `post_requests_total` |
| Avoid `_info` suffix unless it's an info metric | `build_info{version="1.2.3"}` | `build_version` (use info pattern) |
| Histograms: name the thing being measured | `http_request_duration_seconds` | `http_latency_histogram` |
| Be consistent within a service | Always `_seconds` for all durations | Mix of `_seconds`, `_ms`, `_duration` |

---

## Next Steps

| File | Topic |
|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Observability Overview — the three pillars and golden signals |
| [02-LOGGING.md](02-LOGGING.md) | Logging — structured logs, ELK stack, Grafana Loki, Fluentd |
| [03-DISTRIBUTED-TRACING.md](03-DISTRIBUTED-TRACING.md) | Distributed Tracing — Jaeger, Zipkin, sampling strategies |
| [04-OPENTELEMETRY.md](04-OPENTELEMETRY.md) | OpenTelemetry — vendor-neutral instrumentation and Collector |
| [05-ALERTING.md](05-ALERTING.md) | Alerting — Alertmanager rules, routing, and integrations |
| [06-SLOS-AND-SLIS.md](06-SLOS-AND-SLIS.md) | SLOs and SLIs — error budgets and burn rate alerts |
| [07-DASHBOARDS.md](07-DASHBOARDS.md) | Dashboards — Grafana panel design and best practices |
