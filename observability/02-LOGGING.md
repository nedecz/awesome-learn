# Logging

## Table of Contents

1. [Structured vs. Unstructured Logging](#structured-vs-unstructured-logging)
2. [Log Levels](#log-levels)
3. [Structured Logging Examples](#structured-logging-examples)
   - [Go (zerolog)](#go-zerolog)
   - [Python (structlog)](#python-structlog)
   - [Java (Logback + JSON)](#java-logback--json)
   - [Node.js (pino)](#nodejs-pino)
4. [Log Aggregation Patterns](#log-aggregation-patterns)
5. [Logging Pipeline](#logging-pipeline)
6. [ELK Stack](#elk-stack)
7. [Grafana Loki](#grafana-loki)
8. [Fluentd and Fluent Bit](#fluentd-and-fluent-bit)
9. [Log Retention and Rotation](#log-retention-and-rotation)
10. [Log Correlation with Traces](#log-correlation-with-traces)
11. [Security Considerations](#security-considerations)
12. [Next Steps](#next-steps)

---

## Structured vs. Unstructured Logging

### Unstructured Logging

Unstructured logs are plain-text strings written to stdout or a file. They are human-readable but **machine-hostile**: parsing them requires fragile regex, and filtering or aggregating across fields is error-prone.

```
# Unstructured log examples (text format)
2025-01-15 14:32:01 ERROR Failed to process payment for order ord_xyz789: connection timeout after 5000ms
2025-01-15 14:32:02 INFO  Request GET /api/orders completed in 142ms with status 200
2025-01-15 14:32:03 WARN  Retry attempt 2/3 for payment service
```

Problems:
- Field extraction requires custom grok patterns or string parsing
- Different developers format messages differently
- Adding new fields (e.g., `user_id`) requires updating parsers
- Cannot easily aggregate "all ERROR logs from order-service in the last hour"

### Structured Logging

Structured logs are key-value records (usually JSON) where every field has a defined name and type. They are machine-parseable and support filtering, aggregation, and correlation without additional parsing logic.

```json
{"timestamp":"2025-01-15T14:32:01.123Z","level":"error","service":"order-service","trace_id":"4bf92f3577b34da6a3ce929d0e0e4736","span_id":"00f067aa0ba902b7","message":"Failed to process payment","order_id":"ord_xyz789","user_id":"usr_abc123","error":"connection timeout","retry_count":0,"duration_ms":5000}
{"timestamp":"2025-01-15T14:32:02.001Z","level":"info","service":"order-service","trace_id":"9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d","span_id":"aabbccdd11223344","message":"Request completed","method":"GET","path":"/api/orders","status":200,"duration_ms":142}
{"timestamp":"2025-01-15T14:32:03.050Z","level":"warn","service":"payment-service","trace_id":"4bf92f3577b34da6a3ce929d0e0e4736","span_id":"99aabb001122ccdd","message":"Retry attempt","attempt":2,"max_attempts":3}
```

### Why Structured Logging Wins

| Dimension | Unstructured | Structured |
|---|---|---|
| **Machine parsing** | Requires brittle regex | Native — query by field name |
| **Field extraction** | Post-hoc; error-prone | Built-in; fields always present |
| **Log aggregation** | Hard to aggregate across messages | Simple — group by `service`, `error` |
| **Trace correlation** | Add manually with conventions | `trace_id` field links to spans automatically |
| **New fields** | Breaks existing parsers | Backward-compatible; just add the field |
| **Filtering in Loki/ES** | Full-text search only | Field-level filtering (`level=error`) |
| **Performance** | Low serialization cost | Slightly higher (JSON marshaling) — negligible in practice |

---

## Log Levels

| Level | Numeric Value | When to Use | Example Message |
|---|---|---|---|
| **TRACE** | 5 (lowest) | Extremely verbose diagnostic data; function entry/exit, loop iterations. Disabled in production. | `"Entering processOrder loop, iteration 42"` |
| **DEBUG** | 10 | Diagnostic information useful for debugging, not needed in normal operation. Often disabled in production. | `"Cache miss for key orders:usr_abc123"` |
| **INFO** | 20 | Normal application events. Significant lifecycle events, startup, shutdown, successful operations. | `"Order ord_xyz789 created successfully"` |
| **WARN** | 30 | Something unexpected but recoverable happened. The application continues normally. Investigate if frequent. | `"Payment service responded slowly (3200ms); retrying"` |
| **ERROR** | 40 | A significant failure occurred. A request or operation failed. Requires investigation. | `"Failed to save order to database: duplicate key"` |
| **FATAL** | 50 (highest) | Catastrophic failure. The application cannot continue and is about to exit. | `"Failed to bind to port 8080: address already in use"` |

```
  Verbosity
  (most)  TRACE ──────────────────────────────────────────────────────────► Development
           │
          DEBUG ──────────────────────────────────────────────────────────► Debug builds
           │
          INFO  ──────────────────────────────────────────────────────────► Production default
           │
          WARN  ──────────────────────────────────────────────────────────► Always enabled
           │
          ERROR ──────────────────────────────────────────────────────────► Always enabled + alert
           │
  (least) FATAL ──────────────────────────────────────────────────────────► Page on-call immediately
```

---

## Structured Logging Examples

### Go (zerolog)

[zerolog](https://github.com/rs/zerolog) is a zero-allocation structured logger for Go with a fluent API.

```go
// logging.go — zerolog setup and usage
package main

import (
    "context"
    "net/http"
    "os"
    "time"

    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
)

func main() {
    // Configure zerolog: output JSON to stdout, set level to Info
    zerolog.TimeFieldFormat = time.RFC3339Nano
    zerolog.SetGlobalLevel(zerolog.InfoLevel)

    // Add common fields to all log messages via a base logger
    logger := zerolog.New(os.Stdout).
        With().
        Timestamp().
        Str("service", "order-service").
        Str("version", "1.2.3").
        Logger()

    // Basic usage
    logger.Info().
        Str("message", "Service started").
        Int("port", 8080).
        Msg("")

    // Logging an error with context
    err := processOrder("ord_xyz789")
    if err != nil {
        logger.Error().
            Err(err).
            Str("order_id", "ord_xyz789").
            Str("user_id", "usr_abc123").
            Int("retry_count", 0).
            Msg("Failed to process order")
    }

    // Logging with trace context (from OpenTelemetry span)
    ctx := context.Background()
    logWithTraceContext(ctx, logger)
}

// logWithTraceContext extracts trace info and adds it to log entries
func logWithTraceContext(ctx context.Context, logger zerolog.Logger) {
    // In practice, extract from OpenTelemetry span:
    // span := trace.SpanFromContext(ctx)
    // traceID := span.SpanContext().TraceID().String()
    // spanID  := span.SpanContext().SpanID().String()

    traceID := "4bf92f3577b34da6a3ce929d0e0e4736"
    spanID := "00f067aa0ba902b7"

    logger.Info().
        Str("trace_id", traceID).
        Str("span_id", spanID).
        Str("order_id", "ord_xyz789").
        Int64("duration_ms", 142).
        Msg("Order retrieved successfully")
}

func processOrder(orderID string) error { return nil }
```

```bash
go get github.com/rs/zerolog
```

Example output:
```json
{"level":"info","service":"order-service","version":"1.2.3","time":"2025-01-15T14:32:01.123456789Z","port":8080,"message":"Service started"}
{"level":"error","service":"order-service","version":"1.2.3","time":"2025-01-15T14:32:01.456Z","error":"timeout","order_id":"ord_xyz789","user_id":"usr_abc123","retry_count":0,"message":"Failed to process order"}
```

### Python (structlog)

[structlog](https://www.structlog.org/) is a structured logging library for Python that wraps the standard `logging` module.

```python
# logging_setup.py — structlog configuration
import structlog
import logging
import sys

def configure_logging():
    """Configure structlog for JSON output with common processors."""
    shared_processors = [
        structlog.contextvars.merge_contextvars,         # Thread-local context
        structlog.processors.add_log_level,              # Add 'level' field
        structlog.processors.TimeStamper(fmt="iso"),     # ISO 8601 timestamp
        structlog.stdlib.add_logger_name,                # Add 'logger' field
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,            # Format exceptions
    ]

    structlog.configure(
        processors=shared_processors + [
            structlog.processors.JSONRenderer()          # Output as JSON
        ],
        wrapper_class=structlog.BoundLogger,
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(file=sys.stdout),
    )


# app.py — using structlog in application code
import structlog
from logging_setup import configure_logging

configure_logging()
log = structlog.get_logger()

# Add request-scoped context (available on all subsequent log calls in this thread)
structlog.contextvars.clear_contextvars()
structlog.contextvars.bind_contextvars(
    trace_id="4bf92f3577b34da6a3ce929d0e0e4736",
    span_id="00f067aa0ba902b7",
    service="order-service",
    request_id="req_aabbccdd",
)

# Basic logging
log.info("Order created", order_id="ord_xyz789", user_id="usr_abc123", total=99.99)
log.warning("Payment slow", duration_ms=3200, attempt=1)

# Error logging with exception
try:
    raise ValueError("duplicate order")
except ValueError:
    log.error("Failed to save order", order_id="ord_xyz789", exc_info=True)
```

```bash
pip install structlog
```

Example output:
```json
{"trace_id": "4bf92f3577b34da6a3ce929d0e0e4736", "span_id": "00f067aa0ba902b7", "service": "order-service", "request_id": "req_aabbccdd", "event": "Order created", "order_id": "ord_xyz789", "user_id": "usr_abc123", "total": 99.99, "level": "info", "timestamp": "2025-01-15T14:32:01.123456Z"}
```

### Java (Logback + JSON)

```xml
<!-- pom.xml — add logback-classic and logstash-logback-encoder -->
<dependencies>
  <dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.4.14</version>
  </dependency>
  <dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
  </dependency>
</dependencies>
```

```xml
<!-- logback.xml — configure JSON output -->
<configuration>
  <appender name="JSON_STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <!-- Add custom fields to every log line -->
      <customFields>{"service":"order-service","version":"1.2.3"}</customFields>
      <!-- Include MDC fields (trace_id, span_id injected by OTel) -->
      <includeMdcKeyName>trace_id</includeMdcKeyName>
      <includeMdcKeyName>span_id</includeMdcKeyName>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="JSON_STDOUT"/>
  </root>
</configuration>
```

```java
// OrderService.java — SLF4J + MDC for structured fields
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import net.logstash.logback.argument.StructuredArguments;

public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public Order createOrder(String userId, OrderRequest req) {
        // MDC fields are automatically included in every log message in this thread
        MDC.put("user_id", userId);
        MDC.put("request_id", req.getRequestId());

        try {
            log.info("Creating order",
                StructuredArguments.keyValue("product_ids", req.getProductIds()),
                StructuredArguments.keyValue("total", req.getTotal())
            );

            Order order = orderRepository.save(req.toOrder(userId));

            log.info("Order created successfully",
                StructuredArguments.keyValue("order_id", order.getId()),
                StructuredArguments.keyValue("duration_ms", req.elapsed())
            );
            return order;

        } catch (Exception e) {
            log.error("Failed to create order",
                StructuredArguments.keyValue("error", e.getMessage()),
                e
            );
            throw e;
        } finally {
            MDC.clear();
        }
    }
}
```

Example output:
```json
{"@timestamp":"2025-01-15T14:32:01.123Z","level":"INFO","service":"order-service","version":"1.2.3","trace_id":"4bf92f3577b34da6a3ce929d0e0e4736","span_id":"00f067aa0ba902b7","user_id":"usr_abc123","request_id":"req_aabbccdd","message":"Order created successfully","order_id":"ord_xyz789","duration_ms":87}
```

### Node.js (pino)

[pino](https://getpino.io/) is a very low-overhead Node.js logger that outputs JSON by default.

```javascript
// logger.js — pino configuration
const pino = require('pino');

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  base: {
    service: 'order-service',
    version: process.env.npm_package_version || '1.0.0',
    pid: process.pid,
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  formatters: {
    level(label) {
      return { level: label };  // Use string level instead of number
    },
  },
  redact: {
    paths: ['req.headers.authorization', 'body.creditCardNumber'],
    censor: '[REDACTED]',
  },
});

module.exports = logger;


// app.js — Express with pino-http middleware
const express = require('express');
const pinoHttp = require('pino-http');
const logger = require('./logger');

const app = express();

// Automatic HTTP request/response logging with correlation
app.use(pinoHttp({
  logger,
  // Add trace context from OTel headers to request logs
  customProps(req) {
    return {
      trace_id: req.headers['x-trace-id'],
      span_id: req.headers['x-span-id'],
    };
  },
}));

app.get('/api/orders', async (req, res) => {
  const childLogger = req.log.child({ order_count: 42 });

  childLogger.info('Fetching orders', { user_id: req.user?.id });

  try {
    const orders = await orderService.getOrders(req.user.id);
    childLogger.info('Orders fetched', { count: orders.length });
    res.json({ orders });
  } catch (err) {
    req.log.error({ err, user_id: req.user?.id }, 'Failed to fetch orders');
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

```bash
npm install pino pino-http express
```

---

## Log Aggregation Patterns

### 1. Centralized Logging Pattern

All services ship logs to a central aggregator. Simple to operate, but the aggregator is a single point of failure.

```
  Service A ──┐
              │  direct log shipping
  Service B ──┤──────────────────────► Log Aggregator ──► Storage ──► UI
              │  (Fluentd/Promtail)     (Loki/ES)
  Service C ──┘
```

### 2. Distributed Logging with Message Buffer

A message broker (Kafka, Redis) buffers logs between shippers and the aggregator, providing backpressure handling and resilience.

```
  Service A ──┐                          ┌─────────────────┐
              │                          │                 │
  Service B ──┤──► Fluentd/Fluent Bit ──►│  Kafka / Redis  │──► Logstash / Loki ──► Storage
              │    (per-host agent)       │  (message buf)  │
  Service C ──┘                          │                 │
                                         └─────────────────┘

  Benefits:
  ● Absorbs log spikes without dropping data
  ● Aggregator can be restarted without losing logs
  ● Multiple consumers (archival + real-time alerting)
```

### 3. Sidecar Pattern (Kubernetes)

Each Pod has a sidecar container that reads the application's log volume and ships to the aggregator. Keeps log shipping out of the application container.

```
  ┌─────────────────────────────────────────────┐
  │                    Pod                      │
  │                                             │
  │  ┌───────────────┐    ┌──────────────────┐  │
  │  │ App Container │───►│ Sidecar Container│  │
  │  │               │    │  (Fluent Bit)    │  │
  │  │ stdout/stderr │    │                  │──┼──► Loki / ES
  │  │     │         │    │  Reads shared    │  │
  │  │     ▼         │    │  volume /logs    │  │
  │  │  /logs/app.log│    └──────────────────┘  │
  │  └───────────────┘          ▲               │
  │         │                   │               │
  │         └──── shared volume ┘               │
  └─────────────────────────────────────────────┘
```

---

## Logging Pipeline

```
                                         Message
  Application                            Buffer         Log
   (emits logs)  Log Shipper           (optional)    Aggregator     Storage        Query/Viz
  ┌───────────┐  ┌───────────────┐   ┌───────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │           │  │               │   │           │  │          │  │          │  │          │
  │  stdout   │─►│ Fluent Bit /  │──►│   Kafka   │─►│Logstash /│─►│   Loki   │─►│ Grafana  │
  │  /logs/*  │  │ Promtail      │   │   Redis   │  │  Loki    │  │   ES     │  │ Kibana   │
  │           │  │               │   │           │  │ Compactor│  │          │  │          │
  └───────────┘  └───────────────┘   └───────────┘  └──────────┘  └──────────┘  └──────────┘
       │                │                                 │              │              │
       │   Collect &    │  Buffer for backpressure   Parse & enrich   Index &        Query &
       │   forward      │  and replay                & route         compress        visualize
```

---

## ELK Stack

The **ELK Stack** (Elasticsearch, Logstash, Kibana) — often extended to **EFK** (Elasticsearch, Fluentd, Kibana) — is the most widely adopted open-source log aggregation platform.

### Architecture Diagram

```
  ┌────────────┐     ┌────────────┐     ┌───────────────────┐     ┌──────────┐
  │  App Pods  │────►│  Fluentd   │────►│    Logstash       │────►│  Kibana  │
  │  (logs)    │     │  /Filebeat │     │  (parse, enrich,  │     │  (UI)    │
  └────────────┘     └────────────┘     │   route)          │     └──────────┘
                                        └────────┬──────────┘
                                                 │
                                                 ▼
                                        ┌─────────────────────────────────────┐
                                        │           Elasticsearch             │
                                        │  ┌────────┐  ┌────────┐            │
                                        │  │ Node 1 │  │ Node 2 │  ...       │
                                        │  │ (data) │  │ (data) │            │
                                        │  └────────┘  └────────┘            │
                                        │         ┌──────────┐               │
                                        │         │  Master  │               │
                                        │         └──────────┘               │
                                        └─────────────────────────────────────┘
```

### Elasticsearch Configuration

```yaml
# elasticsearch.yml
cluster.name: logs-cluster
node.name: es-node-1
node.roles: [ master, data, ingest ]

# Data paths
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

# Network
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300

# Discovery (single-node for dev, list nodes for production)
discovery.type: single-node
# For production cluster:
# discovery.seed_hosts: ["es-node-1", "es-node-2", "es-node-3"]
# cluster.initial_master_nodes: ["es-node-1", "es-node-2"]

# Security (enable in production)
xpack.security.enabled: false   # Set to true with TLS in production
xpack.security.http.ssl.enabled: false

# JVM heap (set to 50% of available RAM, max 32GB)
# Set in jvm.options: -Xms4g -Xmx4g

# Index settings
action.auto_create_index: true
indices.memory.index_buffer_size: 30%
```

### Logstash Pipeline Configuration

```ruby
# /etc/logstash/conf.d/logs.conf

# INPUT: Receive logs from Beats/Fluentd
input {
  beats {
    port => 5044
    type => "beats"
  }
  # Also accept direct TCP input
  tcp {
    port => 5000
    codec => json_lines
  }
}

# FILTER: Parse, enrich, and normalize log fields
filter {

  # Parse JSON logs (structured logging)
  if [message] =~ /^\{/ {
    json {
      source => "message"
      target => "parsed"
    }
    # Promote parsed fields to top level
    ruby {
      code => "
        if event.get('parsed')
          event.get('parsed').each { |k, v| event.set(k, v) }
          event.remove('parsed')
        end
      "
    }
  }

  # Parse unstructured logs with grok
  if [type] == "nginx" {
    grok {
      match => {
        "message" => '%{IPORHOST:client_ip} - %{DATA:user} \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{URIPATH:path}(?:%{URIPARAM:params})? HTTP/%{NUMBER:http_version}" %{INT:status:int} %{INT:bytes:int}'
      }
      overwrite => ["message"]
    }
    date {
      match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
      target => "@timestamp"
    }
  }

  # Add GeoIP data from client IP
  if [client_ip] {
    geoip {
      source => "client_ip"
      target => "geoip"
    }
  }

  # Normalize log level field name
  if [level] {
    mutate {
      rename => { "level" => "log_level" }
      uppercase => ["log_level"]
    }
  }

  # Drop health check noise
  if [path] == "/health" or [path] == "/metrics" {
    drop {}
  }
}

# OUTPUT: Send to Elasticsearch
output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{[service]}-%{+YYYY.MM.dd}"
    # Use ILM for automatic index lifecycle management
    ilm_enabled => true
    ilm_rollover_alias => "logs"
    ilm_policy => "logs-policy"
  }

  # Also output to stdout for debugging (remove in production)
  # stdout { codec => rubydebug }
}
```

### Kibana Index Pattern and Queries

```
# Index Pattern Setup (via Kibana UI or API)
# 1. Go to Stack Management → Index Patterns → Create index pattern
# 2. Pattern: logs-*
# 3. Time field: @timestamp
# 4. Save and navigate to Discover

# Example Kibana Query Language (KQL) queries:

# All errors from order-service in the last hour
service: "order-service" AND log_level: "ERROR"

# Orders failing with specific error
service: "order-service" AND message: "Failed to process payment"

# All requests slower than 1 second
duration_ms > 1000

# Requests with a specific trace ID (cross-service correlation)
trace_id: "4bf92f3577b34da6a3ce929d0e0e4736"

# High-severity events from any service in last 15 min
log_level: ("ERROR" OR "FATAL") AND @timestamp > now-15m

# 5xx errors with response time
status >= 500 AND duration_ms: *
```

---

## Grafana Loki

Grafana Loki is a log aggregation system designed to be **cost-effective and Kubernetes-native**. Unlike Elasticsearch, Loki does **not** full-text index log content — it only indexes log labels. Log content is compressed and stored in object storage (S3, GCS, filesystem).

### Architecture Diagram

```
  ┌────────────────────────────────────────────────────────────┐
  │                       Grafana Loki                         │
  │                                                            │
  │  ┌─────────────┐    ┌──────────────┐    ┌─────────────┐   │
  │  │ Distributor │───►│   Ingester   │───►│  Compactor  │   │
  │  │             │    │              │    │             │   │
  │  │ Validates & │    │ Writes to    │    │ Compacts &  │   │
  │  │ routes to   │    │ WAL + chunks │    │ deduplicates│   │
  │  │ ingesters   │    │              │    │ old chunks  │   │
  │  └─────────────┘    └──────┬───────┘    └─────────────┘   │
  │         ▲                  │                               │
  │         │                  ▼                               │
  │  ┌──────┴──────┐    ┌──────────────┐    ┌─────────────┐   │
  │  │  Promtail   │    │  Object Store│    │   Querier   │   │
  │  │  (shipper)  │    │  (S3/GCS/FS) │◄───│             │   │
  │  └─────────────┘    └──────────────┘    │ Executes    │   │
  │                                         │ LogQL       │   │
  │                                         └──────┬──────┘   │
  └────────────────────────────────────────────────┼──────────┘
                                                   │
                                                   ▼
                                            ┌─────────────┐
                                            │   Grafana   │
                                            │  (Explore)  │
                                            └─────────────┘
```

### Loki Configuration

```yaml
# loki-config.yaml
auth_enabled: false   # Set to true for multi-tenant

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  log_level: info

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

# Schema configuration
schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

# Retention
limits_config:
  retention_period: 744h   # 31 days
  ingestion_rate_mb: 16
  ingestion_burst_size_mb: 32
  max_label_names_per_series: 30
  max_label_value_length: 2048
  # Important: enforce cardinality limit on labels
  max_streams_per_user: 10000

compactor:
  working_directory: /loki/compactor
  retention_enabled: true
  retention_delete_delay: 2h
```

### LogQL Examples

LogQL is Loki's query language. It has two query types: **log queries** (return log lines) and **metric queries** (return numeric values derived from logs).

```logql
# 1. Log line filter — find all error logs from order-service
{service="order-service"} |= "ERROR"


# 2. Regex filter — find logs mentioning payment timeout
{service="order-service"} |~ "payment.*timeout"


# 3. Label filter — structured log field filter (after JSON parsing)
{service="order-service"} | json | level="error"


# 4. rate() — requests per second from all services (metric query)
rate({job="kubernetes-pods"}[5m])


# 5. count_over_time() — count error log lines in last 1 hour per service
sum by (service) (
  count_over_time({job="kubernetes-pods"} |= "ERROR" [1h])
)


# 6. JSON parsing — extract and filter on JSON fields inline
{service="order-service"}
  | json
  | order_id != ""
  | duration_ms > 1000


# 7. line_format — reformat log output for readability
{service="order-service"}
  | json
  | line_format "{{.timestamp}} [{{.level}}] {{.message}} order={{.order_id}}"


# 8. JSON + label filter + rate — error rate using structured fields
sum by (service) (
  rate(
    {job="kubernetes-pods"}
    | json
    | level="error" [5m]
  )
)
```

### Label Best Practices

| Use as Labels ✅ | Do NOT Use as Labels ❌ |
|---|---|
| `service` — finite, low-cardinality | `user_id` — millions of unique values |
| `namespace` — handful of Kubernetes namespaces | `request_id` — unique per request |
| `pod` — bounded by cluster size | `trace_id` — unique per trace |
| `environment` — dev/staging/prod | `order_id` — unbounded |
| `level` — debug/info/warn/error/fatal | `ip_address` — high cardinality |
| `region` — handful of cloud regions | `timestamp` — never a label; it's the stream key |
| `container` — bounded by workload count | `session_id` — per-user sessions |

---

## Fluentd and Fluent Bit

### Comparison Table

| Dimension | Fluentd | Fluent Bit |
|---|---|---|
| **Language** | Ruby (C core) | C only |
| **Memory footprint** | ~40 MB | ~650 KB |
| **CPU usage** | Moderate | Very low |
| **Plugin ecosystem** | 500+ plugins | ~80 plugins (growing) |
| **Performance** | High | Very high |
| **Kubernetes DaemonSet** | Works but heavy | Ideal — designed for this |
| **Aggregator role** | Common choice | Less common (use Fluentd) |
| **Config format** | XML-like directives | INI-like sections |
| **Best for** | Complex routing, aggregation, enrichment | Log forwarding at the node level |
| **Recommended combo** | Aggregator | DaemonSet shipper → Fluentd aggregator |

### Fluentd Configuration

```xml
<!-- /etc/fluentd/fluent.conf -->

<!-- SOURCE: Collect from Kubernetes container logs -->
<source>
  @type tail
  path /var/log/containers/*.log
  pos_file /var/log/fluentd-containers.log.pos
  tag kubernetes.*
  read_from_head true
  <parse>
    @type cri              <!-- CRI format for containerd/CRI-O -->
  </parse>
</source>

<!-- SOURCE: Also accept logs from Fluent Bit forwarders -->
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<!-- FILTER: Parse JSON log content -->
<filter kubernetes.**>
  @type parser
  key_name log
  reserve_data true
  remove_key_name_field true
  <parse>
    @type json
    time_key timestamp
    time_format %Y-%m-%dT%H:%M:%S.%NZ
  </parse>
</filter>

<!-- FILTER: Add Kubernetes metadata to each log record -->
<filter kubernetes.**>
  @type kubernetes_metadata
  skip_labels false
  skip_container_metadata false
</filter>

<!-- FILTER: Drop health check noise -->
<filter kubernetes.**>
  @type grep
  <exclude>
    key path
    pattern ^/(health|metrics|ready)$
  </exclude>
</filter>

<!-- MATCH: Route to Elasticsearch -->
<match kubernetes.**>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  logstash_prefix logs
  include_tag_key true
  <buffer>
    @type file
    path /var/log/fluentd-buffers/elasticsearch
    flush_mode interval
    flush_interval 5s
    chunk_limit_size 2M
    queue_limit_length 32
    retry_max_interval 30
    retry_forever true
  </buffer>
</match>
```

### Fluent Bit Configuration

```ini
# /etc/fluent-bit/fluent-bit.conf

[SERVICE]
    Flush         5
    Log_Level     info
    Daemon        off
    Parsers_File  parsers.conf
    HTTP_Server   On
    HTTP_Listen   0.0.0.0
    HTTP_Port     2020   # Exposes /api/v1/metrics for Prometheus scraping

[INPUT]
    Name              tail
    Tag               kube.*
    Path              /var/log/containers/*.log
    Parser            cri
    DB                /var/log/flb_kube.db
    Mem_Buf_Limit     50MB
    Skip_Long_Lines   On
    Refresh_Interval  10

[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
    Merge_Log           On        # Merge JSON 'log' field into record
    Keep_Log            Off
    K8S-Logging.Parser  On
    K8S-Logging.Exclude On

[FILTER]
    Name    grep
    Match   kube.*
    Exclude path ^/(health|metrics)

[OUTPUT]
    Name            loki
    Match           kube.*
    Host            loki
    Port            3100
    Labels          job=fluent-bit, node=${NODE_NAME}
    Label_Keys      $kubernetes['namespace_name'],$kubernetes['pod_name'],$kubernetes['container_name']
    Auto_Kubernetes_Labels  On
    Line_Format     json
    Retry_Limit     False
```

### Kubernetes Sidecar Pattern

```yaml
# kubernetes/pod-with-fluent-bit-sidecar.yaml
apiVersion: v1
kind: Pod
metadata:
  name: order-service
spec:
  volumes:
    - name: log-volume
      emptyDir: {}
    - name: fluent-bit-config
      configMap:
        name: fluent-bit-sidecar-config

  containers:
    # Application container — writes logs to shared volume
    - name: order-service
      image: myregistry/order-service:1.2.3
      env:
        - name: LOG_FILE
          value: /var/log/app/order-service.log
      volumeMounts:
        - name: log-volume
          mountPath: /var/log/app

    # Sidecar container — reads logs and ships to Loki
    - name: fluent-bit
      image: fluent/fluent-bit:2.2.0
      resources:
        requests:
          cpu: 10m
          memory: 30Mi
        limits:
          cpu: 100m
          memory: 100Mi
      volumeMounts:
        - name: log-volume
          mountPath: /var/log/app
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc
```

---

## Log Retention and Rotation

### logrotate Configuration

```bash
# /etc/logrotate.d/app-logs
/var/log/myapp/*.log {
    daily                    # Rotate daily
    rotate 7                 # Keep 7 days of rotated logs
    compress                 # Compress rotated logs with gzip
    delaycompress            # Delay compression by one rotation (so current day stays uncompressed)
    missingok                # Don't error if log file is missing
    notifempty               # Don't rotate if file is empty
    create 0640 myapp myapp  # Create new file with these permissions
    sharedscripts            # Run postrotate once even if multiple files match
    postrotate
        # Signal the app to reopen its log files
        kill -HUP $(cat /var/run/myapp.pid 2>/dev/null) 2>/dev/null || true
    endscript
}
```

### Retention Policy Table

| Environment | Log Level | Retention | Storage Tier | Notes |
|---|---|---|---|---|
| Development | DEBUG | 1 day | Local disk | Verbose; short-lived |
| Staging | INFO | 7 days | Local disk / cheap object storage | Match prod log levels |
| Production | INFO | 30 days hot, 1 year cold | Hot: SSD-backed TSDB; Cold: S3/GCS | Typical compliance baseline |
| Production (regulated) | INFO | 7 years | Immutable object storage (WORM) | PCI-DSS, HIPAA, SOX requirements |
| Security/Audit | INFO | 1–7 years | Immutable, tamper-evident | Separate from application logs |

---

## Log Correlation with Traces

Linking log lines to their corresponding distributed trace is one of the highest-value observability improvements you can make. It lets you jump from a Grafana Loki log line directly to the Jaeger trace for that request.

### Go — Injecting Trace Context into Logs

```go
// tracing_log.go — inject OTel trace context into zerolog
package logger

import (
    "context"

    "github.com/rs/zerolog"
    "go.opentelemetry.io/otel/trace"
)

// FromContext returns a zerolog.Logger enriched with the current span's trace/span ID.
func FromContext(ctx context.Context, base zerolog.Logger) zerolog.Logger {
    span := trace.SpanFromContext(ctx)
    if !span.SpanContext().IsValid() {
        return base
    }
    return base.With().
        Str("trace_id", span.SpanContext().TraceID().String()).
        Str("span_id", span.SpanContext().SpanID().String()).
        Bool("trace_sampled", span.SpanContext().IsSampled()).
        Logger()
}

// Usage in a handler:
func (s *OrderService) GetOrder(ctx context.Context, orderID string) (*Order, error) {
    log := logger.FromContext(ctx, s.log)  // Inherits trace_id + span_id

    log.Info().Str("order_id", orderID).Msg("Fetching order")

    order, err := s.repo.FindByID(ctx, orderID)
    if err != nil {
        log.Error().Err(err).Str("order_id", orderID).Msg("Order not found")
        return nil, err
    }
    return order, nil
}
```

### Python — Injecting Trace Context into structlog

```python
# trace_logging.py — inject OTel trace context into structlog
from opentelemetry import trace
import structlog

def add_trace_context(logger, method, event_dict):
    """structlog processor that adds OTel trace/span IDs to every log event."""
    span = trace.get_current_span()
    ctx = span.get_span_context()

    if ctx.is_valid:
        event_dict["trace_id"] = format(ctx.trace_id, "032x")
        event_dict["span_id"] = format(ctx.span_id, "016x")
        event_dict["trace_sampled"] = ctx.trace_flags.sampled

    return event_dict


# Add to structlog processor chain
structlog.configure(
    processors=[
        add_trace_context,           # ← inject trace context
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
)
```

### Grafana: Jump from Log to Trace

Configure Loki data source in Grafana to link trace IDs to Jaeger:

```yaml
# In Grafana data source configuration (via UI or provisioning YAML):
# Data Sources → Loki → Derived fields
# Field name: trace_id
# Regex: "trace_id":"([a-f0-9]+)"
# URL: http://jaeger:16686/trace/$${__value.raw}
# Internal link: Tempo or Jaeger data source
```

---

## Security Considerations

### PII Redaction

Never log personally identifiable information (PII). Use redaction at the logging layer.

```go
// Go — redact sensitive fields before logging
func redact(value string) string {
    if len(value) <= 4 {
        return "****"
    }
    return value[:2] + strings.Repeat("*", len(value)-4) + value[len(value)-2:]
}

log.Info().
    Str("email", redact(user.Email)).         // "jo****@example.com" → "jo****le"
    Str("card_last4", order.CardLast4).        // OK — only last 4
    // NEVER: Str("card_number", order.CardNumber)
    Msg("Order placed")
```

```python
# Python — pino-style redact using structlog processor
REDACTED_FIELDS = {"password", "credit_card", "ssn", "api_key", "token", "secret"}

def redact_sensitive_fields(logger, method, event_dict):
    for key in list(event_dict.keys()):
        if key.lower() in REDACTED_FIELDS:
            event_dict[key] = "[REDACTED]"
    return event_dict
```

### Audit Log Format

Audit logs capture security-relevant events and must meet higher standards than application logs.

```json
{
  "timestamp": "2025-01-15T14:32:01.123Z",
  "event_type": "RESOURCE_MODIFIED",
  "actor": {
    "user_id": "usr_abc123",
    "email": "admin@example.com",
    "ip_address": "203.0.113.42",
    "user_agent": "Mozilla/5.0 ..."
  },
  "resource": {
    "type": "order",
    "id": "ord_xyz789",
    "action": "status_changed"
  },
  "changes": {
    "before": { "status": "pending" },
    "after":  { "status": "cancelled" }
  },
  "result": "SUCCESS",
  "request_id": "req_aabbccdd",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736"
}
```

### What to Never Log

| Category | Examples | Why |
|---|---|---|
| **Authentication credentials** | Passwords, API keys, JWT tokens, OAuth tokens | Credential exposure → account takeover |
| **Payment data** | Full credit card numbers, CVV, bank account numbers | PCI-DSS violation |
| **Personal identifiers** | SSN, passport numbers, national IDs | Privacy law violations (GDPR, CCPA) |
| **Health information** | Diagnoses, medication, medical records | HIPAA violation |
| **Private keys** | TLS certificates, SSH private keys | Complete security compromise |
| **Full request bodies** | May contain any of the above | Unpredictable content; redact or omit |
| **Encryption keys** | AES keys, HMAC secrets | Data could be decrypted |
| **Internal infrastructure** | Internal IP ranges, internal hostnames in external logs | Reduces attack surface visibility |

---

## Next Steps

| File | Topic |
|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Observability Overview — three pillars and golden signals |
| [01-METRICS.md](01-METRICS.md) | Metrics — Prometheus, PromQL, and Grafana dashboards |
| [03-DISTRIBUTED-TRACING.md](03-DISTRIBUTED-TRACING.md) | Distributed Tracing — Jaeger, Zipkin, context propagation |
| [04-OPENTELEMETRY.md](04-OPENTELEMETRY.md) | OpenTelemetry — unified SDK for all three pillars |
| [05-ALERTING.md](05-ALERTING.md) | Alerting — Alertmanager, alert rules, on-call routing |
| [06-SLOS-AND-SLIS.md](06-SLOS-AND-SLIS.md) | SLOs and SLIs — error budgets, burn rates |
| [09-ANTI-PATTERNS.md](09-ANTI-PATTERNS.md) | Anti-Patterns — logging anti-patterns and how to fix them |
