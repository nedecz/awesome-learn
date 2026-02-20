# Connection Management

## Table of Contents

1. [Overview](#overview)
   - [Why Connection Management Matters](#why-connection-management-matters)
   - [The Cost of a Database Connection](#the-cost-of-a-database-connection)
   - [Connection Lifecycle](#connection-lifecycle)
2. [Connection Pooling](#connection-pooling)
   - [Why Pooling Is Essential](#why-pooling-is-essential)
   - [Pool Sizing](#pool-sizing)
   - [Pool Configuration Parameters](#pool-configuration-parameters)
   - [Connection Validation](#connection-validation)
3. [Connection Pool Implementations](#connection-pool-implementations)
   - [HikariCP (Java)](#hikaricp-java)
   - [Npgsql (C# / .NET)](#npgsql-c--net)
   - [psycopg2 and asyncpg (Python)](#psycopg2-and-asyncpg-python)
   - [database/sql (Go)](#databasesql-go)
   - [node-postgres (Node.js)](#node-postgres-nodejs)
   - [Implementation Comparison](#implementation-comparison)
4. [External Connection Proxies](#external-connection-proxies)
   - [PgBouncer](#pgbouncer)
   - [ProxySQL (MySQL)](#proxysql-mysql)
   - [PgCat](#pgcat)
   - [Amazon RDS Proxy](#amazon-rds-proxy)
   - [Proxy vs Application-Level Pooling](#proxy-vs-application-level-pooling)
5. [Timeouts](#timeouts)
   - [Timeout Types](#timeout-types)
   - [PostgreSQL Timeout Configuration](#postgresql-timeout-configuration)
   - [MySQL Timeout Configuration](#mysql-timeout-configuration)
6. [Resilience Patterns](#resilience-patterns)
   - [Retry with Exponential Backoff](#retry-with-exponential-backoff)
   - [Circuit Breaker for Database Connections](#circuit-breaker-for-database-connections)
   - [Health Checks](#health-checks)
   - [Failover Handling](#failover-handling)
7. [Connection Management in Serverless](#connection-management-in-serverless)
   - [The Serverless Connection Problem](#the-serverless-connection-problem)
   - [RDS Proxy for Lambda](#rds-proxy-for-lambda)
   - [Neon Pooler](#neon-pooler)
   - [PlanetScale Connection Strings](#planetscale-connection-strings)
   - [Why Traditional Pooling Fails in Serverless](#why-traditional-pooling-fails-in-serverless)
8. [Monitoring Connections](#monitoring-connections)
   - [PostgreSQL: pg_stat_activity](#postgresql-pg_stat_activity)
   - [MySQL: SHOW PROCESSLIST](#mysql-show-processlist)
   - [Connection Count Metrics](#connection-count-metrics)
   - [Pool Exhaustion Alerting](#pool-exhaustion-alerting)
9. [Version History](#version-history)

---

## Overview

### Why Connection Management Matters

Every interaction between an application and a database happens over a **connection** — a stateful, authenticated session between a client and the database server. How you create, reuse, and retire these connections directly impacts latency, stability, and cost.

Poor connection management is one of the most common causes of production database incidents. Opening too many connections overwhelms the server; opening too few starves the application. Leaked connections silently drain the pool until the application halts.

### The Cost of a Database Connection

A database connection is not lightweight. Each consumes real server resources regardless of whether it is executing a query.

| Resource | PostgreSQL (per connection) | MySQL (per connection) |
|---|---|---|
| **Memory** | ~5–10 MB (work_mem, temp buffers, process stack) | ~1–4 MB (thread stack, buffers) |
| **Process / Thread** | Dedicated OS process (fork of postmaster) | Dedicated OS thread |
| **File descriptors** | Multiple FDs per connection | Multiple FDs per connection |
| **Authentication** | TLS handshake + SCRAM/MD5 auth per connect | TLS handshake + auth plugin per connect |

For PostgreSQL, each new connection forks a backend process. At 300 connections, the server manages 300 OS processes, consuming 1.5–3 GB just for connection overhead.

```
  PostgreSQL Process Model
  ────────────────────────

  ┌────────────────┐
  │   postmaster   │  (main process — listens on port 5432)
  └───────┬────────┘
          │ fork()
    ┌─────┼──────────────────────┐
    ▼     ▼          ▼           ▼
  ┌────┐ ┌────┐   ┌────┐     ┌────┐
  │back│ │back│   │back│ ... │back│   N backend processes
  │end │ │end │   │end │     │end │   ≈ 5-10 MB each
  │ 1  │ │ 2  │   │ 3  │     │ N  │
  └────┘ └────┘   └────┘     └────┘
```

### Connection Lifecycle

```
  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ 1. DNS   │───▶│ 2. TCP   │───▶│ 3. TLS   │───▶│ 4. Auth  │───▶│ 5. Ready │
  │ Resolve  │    │ Handshake│    │ Handshake│    │ Exchange │    │ to Query │
  │ ~1 ms    │    │ ~1-3 ms  │    │ ~5-15 ms │    │ ~3-5 ms  │    │          │
  └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘

  Total setup: 10–25 ms same region, 50–150 ms cross-region with TLS
```

Without pooling, this setup cost is paid for **every query**. A page running 5 queries spends 50–750 ms on connection overhead alone.

---

## Connection Pooling

### Why Pooling Is Essential

A **connection pool** maintains pre-established database connections that application threads borrow and return. Instead of paying the connection setup cost per query, you pay it once when the pool initializes.

### Pool Sizing

One of the most common mistakes is making the pool too large. The widely-referenced formula from the HikariCP wiki:

```
  connections = (core_count × 2) + effective_spindle_count
```

- **core_count** — CPU cores on the database server (not the app server)
- **effective_spindle_count** — number of spinning hard drives (for SSDs, use 1)

For a database server with 4 CPU cores and SSD storage:

```
  connections = (4 × 2) + 1 = 9
```

| Database Server Cores | SSD | Recommended Pool Size |
|---|---|---|
| 2 | Yes | 5 |
| 4 | Yes | 9 |
| 8 | Yes | 17 |
| 16 | Yes | 33 |

> **Rule of thumb**: If you have multiple app instances, divide the pool across them. With 4 app servers pointing at a 4-core database, each app should have a pool of ~2–3 connections.

### Pool Configuration Parameters

| Parameter | Description | Recommended |
|---|---|---|
| **minimumIdle** | Minimum connections kept open when idle | Equal to max for stable workloads |
| **maximumPoolSize** | Maximum connections the pool will create | Use the formula above |
| **connectionTimeout** | Time to wait for a connection from the pool | 5–10s (fail fast) |
| **idleTimeout** | Time before an idle connection is retired | 5–10 min |
| **maxLifetime** | Maximum total lifetime of a connection | Slightly less than DB `wait_timeout` |
| **validationTimeout** | Time allowed for a connection health check | 3–5s |

### Connection Validation

Connections can go stale — the database may close them, or a network blip may sever them.

| Validation Strategy | Tradeoff |
|---|---|
| **Test on borrow** | Adds latency to every checkout |
| **Test on return** | Bad connections sit in pool longer |
| **Background validation** | Minimal user-facing latency; preferred approach |
| **Connection max age** | Simple; no per-query overhead |

---

## Connection Pool Implementations

### HikariCP (Java)

HikariCP is the default connection pool for Spring Boot and the fastest JDBC pool available.

```yaml
# application.yml (Spring Boot)
spring:
  datasource:
    url: jdbc:postgresql://db-host:5432/mydb
    username: app_user
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 10
      connection-timeout: 5000
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 30000
```

### Npgsql (C# / .NET)

Npgsql includes a built-in connection pool, enabled by default through the connection string.

```csharp
var connectionString =
    "Host=db-host;Port=5432;Database=mydb;" +
    "Username=app_user;Password=secret;" +
    "Minimum Pool Size=5;Maximum Pool Size=10;" +
    "Connection Idle Lifetime=300;Timeout=5";

await using var conn = await dataSource.OpenConnectionAsync();
await using var cmd = new NpgsqlCommand("SELECT id, name FROM users", conn);
await using var reader = await cmd.ExecuteReaderAsync();
```

### psycopg2 and asyncpg (Python)

```python
# psycopg2 — synchronous
from psycopg2 import pool

connection_pool = pool.ThreadedConnectionPool(
    minconn=2, maxconn=10,
    host="db-host", dbname="mydb",
    user="app_user", password="secret",
)

conn = connection_pool.getconn()
try:
    with conn.cursor() as cur:
        cur.execute("SELECT id, name FROM users")
        rows = cur.fetchall()
finally:
    connection_pool.putconn(conn)
```

```python
# asyncpg — high-performance async driver
pool = await asyncpg.create_pool(
    host="db-host", database="mydb",
    user="app_user", password="secret",
    min_size=2, max_size=10,
    max_inactive_connection_lifetime=300.0,
)

async with pool.acquire() as conn:
    rows = await conn.fetch("SELECT id, name FROM users")
```

### database/sql (Go)

Go's `database/sql` package includes a built-in connection pool.

```go
db, err := sql.Open("postgres",
    "host=db-host port=5432 dbname=mydb user=app_user password=secret sslmode=require")
if err != nil {
    log.Fatal(err)
}

db.SetMaxOpenConns(10)
db.SetMaxIdleConns(10)
db.SetConnMaxLifetime(30 * time.Minute)
db.SetConnMaxIdleTime(5 * time.Minute)
```

### node-postgres (Node.js)

```javascript
const { Pool } = require('pg');

const pool = new Pool({
  host: 'db-host',
  database: 'mydb',
  user: 'app_user',
  password: process.env.DB_PASSWORD,
  max: 10,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
});

const { rows } = await pool.query('SELECT id, name FROM users');
```

### Implementation Comparison

| Feature | HikariCP | Npgsql | asyncpg | database/sql | node-postgres |
|---|---|---|---|---|---|
| **Language** | Java | C# / .NET | Python | Go | JavaScript |
| **Async support** | No (blocking JDBC) | Yes | Yes (native) | Yes (goroutines) | Yes (Promises) |
| **Validation** | JDBC4 isValid() | Background pruning | Automatic | Automatic | Manual ping |
| **Leak detection** | Yes | No | No | No | No |
| **Default max pool** | 10 | 100 | 10 | Unlimited | 10 |

---

## External Connection Proxies

An **external connection proxy** multiplexes thousands of client connections onto a small number of server connections.

```
  ┌─────┐
  │App 1│──10──┐
  └─────┘      │
  ┌─────┐      ▼
  │App 2│──10──▶ PgBouncer ──10──▶ PostgreSQL
  └─────┘      ▲  (1000→10)       (10 server conns)
  ┌─────┐      │
  │App 3│──10──┘
  └─────┘
```

### PgBouncer

PgBouncer is the most widely used connection proxy for PostgreSQL. It supports three pooling modes:

| Mode | Description | Limitations |
|---|---|---|
| **Transaction** | Connection assigned for duration of a transaction | No SET, LISTEN, or session-level prepared statements* |
| **Session** | Connection assigned for duration of client session | Minimal multiplexing benefit |
| **Statement** | Connection assigned per statement | No multi-statement transactions |

\* PgBouncer 1.21+ supports protocol-level prepared statements in transaction mode.

```ini
; pgbouncer.ini
[databases]
mydb = host=db-host port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
reserve_pool_size = 5
server_idle_timeout = 600
server_lifetime = 3600
```

### ProxySQL (MySQL)

ProxySQL provides connection pooling, query routing, and read/write splitting for MySQL.

```sql
-- ProxySQL admin: configure servers and read/write splitting
INSERT INTO mysql_servers (hostgroup_id, hostname, port, max_connections)
VALUES
  (0, 'primary.db.internal', 3306, 100),
  (1, 'replica1.db.internal', 3306, 100);

INSERT INTO mysql_query_rules (rule_id, match_pattern, destination_hostgroup)
VALUES (1, '^SELECT', 1);

LOAD MYSQL SERVERS TO RUNTIME;
LOAD MYSQL QUERY RULES TO RUNTIME;
```

### PgCat

PgCat is a PostgreSQL proxy written in Rust, designed as a modern alternative to PgBouncer. Key advantages: multi-threaded (Tokio async runtime), built-in sharding support, load balancing across replicas, and a Prometheus `/metrics` endpoint.

### Amazon RDS Proxy

RDS Proxy is a fully managed connection proxy from AWS. Key features: IAM authentication (no passwords in code), connection pinning awareness, automatic failover, and up to 50% reduction in Aurora failover time.

### Proxy vs Application-Level Pooling

| Consideration | Application Pool | External Proxy |
|---|---|---|
| **Best for** | Single monolith, few instances | Microservices, serverless |
| **Multiplexing** | Per-instance only | Across all clients globally |
| **Read/write splitting** | Application logic | Automatic via query routing |
| **Failover** | Application must handle | Proxy handles transparently |
| **Latency** | Zero hop (in-process) | Extra network hop (~0.5–1 ms) |
| **Operational cost** | None | Must monitor and maintain |

> **Guideline**: Start with application-level pooling. Add an external proxy when you have more than ~5 application instances or when serverless functions connect directly.

---

## Timeouts

Timeouts are critical guardrails that prevent a single slow query or network issue from cascading into a full outage.

### Timeout Types

| Timeout | What It Controls | Risk If Missing |
|---|---|---|
| **Connection timeout** | Max time to establish a new connection | App hangs waiting for unreachable DB |
| **Statement timeout** | Max time a single query can run | Long queries block connections and consume resources |
| **Idle timeout** | Time before an unused connection is closed | Stale connections waste server resources |
| **Lock wait timeout** | Max time to wait for a row or table lock | Deadlocks and lock chains cascade |
| **Idle in transaction** | Max time a transaction can sit idle | Holds locks, blocks autovacuum (PostgreSQL) |

### PostgreSQL Timeout Configuration

```sql
-- postgresql.conf or ALTER SYSTEM
statement_timeout = '30s';
idle_in_transaction_session_timeout = '60s';
lock_timeout = '10s';
authentication_timeout = '10s';
tcp_keepalives_idle = 60;
tcp_keepalives_interval = 10;
tcp_keepalives_count = 3;
```

```sql
-- Per-transaction override
BEGIN;
SET LOCAL statement_timeout = '60s';
SELECT * FROM generate_large_report();
COMMIT;  -- reverts to session default
```

### MySQL Timeout Configuration

```sql
-- my.cnf / SET GLOBAL
connect_timeout = 10;
wait_timeout = 600;
interactive_timeout = 3600;
innodb_lock_wait_timeout = 10;
max_execution_time = 30000;   -- milliseconds (MySQL 5.7.8+)
net_read_timeout = 30;
net_write_timeout = 60;
```

---

## Resilience Patterns

### Retry with Exponential Backoff

Transient failures — network blips, pool exhaustion, primary failover — are common. Exponential backoff with jitter prevents thundering herds.

```python
import random, time, psycopg2

def execute_with_retry(pool, query, params=None, max_retries=3):
    for attempt in range(max_retries):
        conn = None
        try:
            conn = pool.getconn()
            with conn.cursor() as cur:
                cur.execute(query, params)
                conn.commit()
                return cur.fetchall()
        except psycopg2.OperationalError:
            if attempt == max_retries - 1:
                raise
            sleep_time = (2 ** attempt) + random.uniform(0, 1)
            time.sleep(sleep_time)
        finally:
            if conn:
                pool.putconn(conn)
```

```
  Attempt 1: fail → wait ~1s   (2^0 + jitter)
  Attempt 2: fail → wait ~2s   (2^1 + jitter)
  Attempt 3: fail → wait ~4s   (2^2 + jitter)
  Attempt 4: success ✓
```

### Circuit Breaker for Database Connections

A circuit breaker prevents an application from repeatedly hitting a database that is down.

```
  ┌────────┐  failure threshold   ┌────────┐  timeout     ┌───────────┐
  │ CLOSED │─────exceeded────────▶│  OPEN  │───expires───▶│ HALF-OPEN │
  │ Normal │◀──────success────────│ Reject │◀──failure────│ Allow one │
  │ traffic│  (from half-open)    │  all   │              │ test call │
  └────────┘                      └────────┘              └───────────┘
```

### Health Checks

```go
func healthHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()

    if err := db.PingContext(ctx); err != nil {
        w.WriteHeader(http.StatusServiceUnavailable)
        json.NewEncoder(w).Encode(map[string]string{"status": "unhealthy", "database": err.Error()})
        return
    }
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "healthy", "database": "connected"})
}
```

### Failover Handling

| Strategy | Recovery Time |
|---|---|
| **DNS-based failover** | 30–300s (depends on TTL) |
| **Proxy-based failover** | 1–30s |
| **Client-side multi-host** | Near-instant with health checks |
| **Cluster-aware driver** | Near-instant |

```
# PostgreSQL libpq multi-host connection string
postgresql://user:pass@primary:5432,standby1:5432/mydb?target_session_attrs=read-write
```

---

## Connection Management in Serverless

### The Serverless Connection Problem

Serverless functions create a fundamentally different connection pattern. Each invocation may run in a separate container with its own pool, and the platform can scale to thousands of containers in seconds.

```
  Traditional: 1 server → 1 pool → 10 conns

  Serverless:  1000 Lambda invocations
               → 1000 containers → 1000 pools → 1000+ connections
               → DATABASE OVERWHELMED
```

### RDS Proxy for Lambda

```python
import boto3, psycopg2

def get_connection():
    token = boto3.client('rds').generate_db_auth_token(
        DBHostname='myproxy.proxy-xxxx.us-east-1.rds.amazonaws.com',
        Port=5432, DBUsername='lambda_user',
    )
    return psycopg2.connect(
        host='myproxy.proxy-xxxx.us-east-1.rds.amazonaws.com',
        port=5432, database='mydb', user='lambda_user',
        password=token, sslmode='require',
    )

conn = None  # reuse across warm invocations

def handler(event, context):
    global conn
    if conn is None or conn.closed:
        conn = get_connection()
    with conn.cursor() as cur:
        cur.execute("SELECT id, name FROM users WHERE id = %s", (event['user_id'],))
        return cur.fetchone()
```

### Neon Pooler

Neon includes a built-in PgBouncer-based pooler. Add `-pooler` to the hostname:```
# Direct (migrations)
postgresql://user:pass@ep-xxxx.us-east-1.aws.neon.tech/mydb

# Pooled (application)
postgresql://user:pass@ep-xxxx-pooler.us-east-1.aws.neon.tech/mydb
```

### PlanetScale Connection Strings

PlanetScale handles pooling at the infrastructure level. Their serverless driver uses HTTP, avoiding TCP overhead:

```javascript
import { connect } from '@planetscale/database';

const conn = connect({
  host: process.env.DATABASE_HOST,
  username: process.env.DATABASE_USERNAME,
  password: process.env.DATABASE_PASSWORD,
});
const results = await conn.execute('SELECT * FROM users WHERE id = ?', [1]);
```

### Why Traditional Pooling Fails in Serverless

| Problem | Explanation |
|---|---|
| **Pool per container** | Each container creates its own pool; can't share across containers |
| **Cold starts** | New containers must establish connections from scratch |
| **No graceful shutdown** | Containers are frozen/thawed without warning |
| **Unpredictable concurrency** | Platform may scale from 1 to 1000 containers instantly |
| **Idle waste** | Warm containers hold connections open even when idle |

**Solution**: Always place a connection proxy (RDS Proxy, PgBouncer, Neon Pooler) between serverless functions and the database.

---

## Monitoring Connections

### PostgreSQL: pg_stat_activity

```sql
-- Count connections by state
SELECT state, count(*)
FROM pg_stat_activity
WHERE backend_type = 'client backend'
GROUP BY state;

-- Find long-running queries (> 60 seconds)
SELECT pid, now() - query_start AS duration, state, query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > interval '60 seconds';

-- Terminate a problematic connection
SELECT pg_terminate_backend(12345);
```

### MySQL: SHOW PROCESSLIST

```sql
SELECT id, user, host, db, command, time, state, info
FROM information_schema.processlist
WHERE command != 'Sleep'
ORDER BY time DESC;
```

### Connection Count Metrics

| Metric | Source | Alert Threshold |
|---|---|---|
| **Total connections** | `pg_stat_activity` / `SHOW STATUS` | > 80% of `max_connections` |
| **Active connections** | state = 'active' | > CPU core count |
| **Idle in transaction** | state = 'idle in transaction' | > 0 for extended periods |
| **Pool active count** | Pool metrics | > 80% of pool max |
| **Pool pending requests** | Pool metrics | > 0 sustained |
| **Connection wait time** | Pool metrics | > 100 ms average |

### Pool Exhaustion Alerting

```yaml
# Prometheus alerting rules
groups:
  - name: database_connection_alerts
    rules:
      - alert: ConnectionPoolNearlyExhausted
        expr: (hikaricp_connections_active / hikaricp_connections_max) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Connection pool above 80% utilization"

      - alert: ConnectionPoolExhausted
        expr: hikaricp_connections_pending > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Requests waiting for database connections"

      - alert: IdleInTransactionTooLong
        expr: pg_stat_activity_max_tx_duration{state="idle in transaction"} > 60
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Transaction idle > 60s — may block vacuums"
```

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial connection management documentation |
