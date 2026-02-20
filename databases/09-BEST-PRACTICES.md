# Best Practices for Production

## Table of Contents

1. [Overview](#overview)
   - [Why Best Practices Matter](#why-best-practices-matter)
   - [The Pillars of Production Readiness](#the-pillars-of-production-readiness)
2. [Backup and Recovery](#backup-and-recovery)
   - [RTO and RPO](#rto-and-rpo)
   - [Logical Backups](#logical-backups)
   - [Physical Backups](#physical-backups)
   - [Point-in-Time Recovery (PITR)](#point-in-time-recovery-pitr)
   - [Backup Verification](#backup-verification)
   - [Backup Rotation Strategies](#backup-rotation-strategies)
3. [High Availability](#high-availability)
   - [Primary-Replica Failover](#primary-replica-failover)
   - [Automatic Failover Tools](#automatic-failover-tools)
   - [DNS and Floating IP Failover](#dns-and-floating-ip-failover)
   - [Split-Brain Prevention](#split-brain-prevention)
4. [Monitoring and Alerting](#monitoring-and-alerting)
   - [Key Metrics to Monitor](#key-metrics-to-monitor)
   - [PostgreSQL: pg_stat_statements](#postgresql-pg_stat_statements)
   - [MySQL: Slow Query Log](#mysql-slow-query-log)
   - [Prometheus Exporters](#prometheus-exporters)
   - [Alerting Rules](#alerting-rules)
5. [Security](#security)
   - [Principle of Least Privilege](#principle-of-least-privilege)
   - [Encrypted Connections (TLS)](#encrypted-connections-tls)
   - [Encryption at Rest](#encryption-at-rest)
   - [Audit Logging](#audit-logging)
   - [SQL Injection Prevention](#sql-injection-prevention)
   - [Network Segmentation](#network-segmentation)
6. [Capacity Planning](#capacity-planning)
   - [Disk Growth Forecasting](#disk-growth-forecasting)
   - [Connection Limits](#connection-limits)
   - [Memory Sizing](#memory-sizing)
   - [IOPS Planning](#iops-planning)
7. [Maintenance Operations](#maintenance-operations)
   - [PostgreSQL: VACUUM and ANALYZE](#postgresql-vacuum-and-analyze)
   - [MySQL: OPTIMIZE TABLE](#mysql-optimize-table)
   - [Index Maintenance](#index-maintenance)
   - [Bloat Management](#bloat-management)
8. [Schema Management](#schema-management)
   - [Migration Tools](#migration-tools)
   - [DDL Change Review Process](#ddl-change-review-process)
   - [Staging Environment Testing](#staging-environment-testing)
9. [Performance Tuning Checklist](#performance-tuning-checklist)
   - [Connection Pooling](#connection-pooling)
   - [Index Strategy](#index-strategy)
   - [Query Review](#query-review)
   - [Configuration Tuning](#configuration-tuning)
10. [Disaster Recovery](#disaster-recovery)
    - [DR Site Setup](#dr-site-setup)
    - [Cross-Region Replication](#cross-region-replication)
    - [Regular DR Drills](#regular-dr-drills)
    - [Runbook Documentation](#runbook-documentation)
11. [Production Readiness Checklist](#production-readiness-checklist)
12. [Version History](#version-history)

---

## Overview

### Why Best Practices Matter

Running a database in development is forgiving. Running one in production is not. A missed backup, an unmonitored replica, or an untuned query can cascade into hours of downtime and permanent data loss. This document consolidates the operational practices that separate a reliable production database from a ticking time bomb.

As Martin Kleppmann writes in *Designing Data-Intensive Applications* (DDIA), Chapter 1:

> *"Reliability means making systems work correctly, even when faults occur."*

Best practices are not about preventing every fault — they are about designing your operations so that faults do not become failures.

### The Pillars of Production Readiness

```
┌──────────────────────────────────────────────────────────────────┐
│                   PRODUCTION-READY DATABASE                      │
├──────────┬──────────┬──────────┬──────────┬──────────────────────┤
│  Backup  │   High   │ Monitor  │ Security │     Maintenance      │
│    &     │  Avail-  │    &     │    &     │         &            │
│ Recovery │  ability │  Alert   │ Access   │  Capacity Planning   │
├──────────┴──────────┴──────────┴──────────┴──────────────────────┤
│                    Schema Management                             │
├─────────────────────────────────────────────────────────────────-┤
│                 Performance Tuning                               │
├──────────────────────────────────────────────────────────────────┤
│                 Disaster Recovery                                │
└──────────────────────────────────────────────────────────────────┘
```

Each pillar reinforces the others. Monitoring detects the failures that high availability mitigates. Backups recover what HA cannot. Security protects the data that all other practices keep available.

---

## Backup and Recovery

Data loss is permanent. Every other failure — slow queries, downtime, even security breaches — can be recovered from if you have your data. Backups are the single most important operational practice.

### RTO and RPO

Before choosing a backup strategy, define your recovery objectives:

| Metric | Definition | Question It Answers |
|---|---|---|
| **RPO** (Recovery Point Objective) | Maximum acceptable data loss measured in time | "How much data can we afford to lose?" |
| **RTO** (Recovery Time Objective) | Maximum acceptable time to restore service | "How long can the database be down?" |

```
  Data Loss Window                  Downtime Window
◀────────────────▶               ◀──────────────────▶

───────────────────────────────────────────────────────▶ time
        ▲               ▲                   ▲
    Last Backup      Failure            Service
                                       Restored

   RPO = time between               RTO = time between
   last backup and failure           failure and restore
```

| RPO Target | Backup Strategy |
|---|---|
| Hours | Periodic logical backups (pg_dump, mysqldump) |
| Minutes | Physical backups + continuous WAL/binlog archiving |
| Seconds | Synchronous replication to standby |
| Zero | Synchronous multi-node commit (distributed consensus) |

### Logical Backups

Logical backups export data as SQL statements or structured formats. They are portable, human-readable, and version-independent — but slow for large databases.

**PostgreSQL — pg_dump:**

```sql
-- Full database dump (custom format, compressed)
pg_dump -Fc -Z 6 -f /backups/mydb_$(date +%Y%m%d_%H%M%S).dump mydb

-- Specific tables only
pg_dump -Fc -t orders -t order_items -f /backups/orders.dump mydb

-- Schema-only dump (useful for migration review)
pg_dump --schema-only -f /backups/mydb_schema.sql mydb

-- Parallel dump for large databases (4 workers)
pg_dump -Fd -j 4 -f /backups/mydb_dir/ mydb
```

**MySQL — mysqldump:**

```sql
-- Full database dump with routines and triggers
mysqldump --single-transaction --routines --triggers \
  --set-gtid-purged=OFF mydb > /backups/mydb_$(date +%Y%m%d).sql

-- All databases
mysqldump --single-transaction --all-databases \
  --master-data=2 > /backups/full_$(date +%Y%m%d).sql

-- Specific tables
mysqldump --single-transaction mydb orders order_items \
  > /backups/orders.sql
```

| Feature | pg_dump | mysqldump |
|---|---|---|
| Consistent snapshot | ✅ Uses transaction | ✅ With `--single-transaction` (InnoDB) |
| Parallel dump | ✅ `-j` workers (directory format) | ❌ Single-threaded |
| Compression | ✅ Built-in (`-Z`) | ❌ Pipe to gzip |
| Large databases (>100 GB) | Slow — prefer physical | Slow — prefer xtrabackup |
| Cross-version restore | ✅ | ✅ |

### Physical Backups

Physical backups copy the raw data files. They are faster to take and restore for large databases but are tied to the same database major version.

**PostgreSQL — pg_basebackup:**

```bash
# Full base backup with WAL streaming
pg_basebackup -D /backups/base_$(date +%Y%m%d) \
  -Ft -z -Xs -P -c fast

# Explanation of flags:
#  -Ft   tar format
#  -z    gzip compression
#  -Xs   stream WAL during backup (ensures consistency)
#  -P    show progress
#  -c fast   issue fast checkpoint to start sooner
```

**MySQL — Percona XtraBackup:**

```bash
# Full backup
xtrabackup --backup --target-dir=/backups/full_$(date +%Y%m%d)

# Incremental backup (based on previous full)
xtrabackup --backup --target-dir=/backups/inc_$(date +%Y%m%d) \
  --incremental-basedir=/backups/full_20250101

# Prepare for restore (apply log)
xtrabackup --prepare --target-dir=/backups/full_20250101
```

| Feature | pg_basebackup | Percona XtraBackup |
|---|---|---|
| Online backup | ✅ | ✅ |
| Incremental | ❌ (full only; use pgBackRest for incremental) | ✅ Built-in |
| Compression | ✅ | ✅ |
| Restore speed | Fast — copy files back | Fast — copy files back |
| PITR support | ✅ With WAL archiving | ✅ With binlog archiving |

### Point-in-Time Recovery (PITR)

PITR lets you restore to any moment between a base backup and the failure. It works by replaying the database's write-ahead log (WAL in PostgreSQL, binlog in MySQL) on top of a physical backup.

```
  ┌──────────────┐    ┌─────────┐   ┌─────────┐   ┌─────────┐
  │  Base Backup  │ + │  WAL 1  │ + │  WAL 2  │ + │  WAL 3  │
  │  (Sunday 2am) │    │ Sun-Mon │   │ Mon-Tue │   │ Tue-Wed │
  └──────────────┘    └─────────┘   └─────────┘   └─────────┘
                                                        │
                                                        ▼
                                           Restore to Tuesday 14:35
```

**PostgreSQL PITR setup (`postgresql.conf`):**

```
# Enable WAL archiving
archive_mode = on
archive_command = 'cp %p /wal_archive/%f'
wal_level = replica
```

**PostgreSQL PITR restore (`postgresql.conf` on recovery target):**

```
restore_command = 'cp /wal_archive/%f %p'
recovery_target_time = '2025-03-15 14:35:00'
recovery_target_action = 'promote'
```

**MySQL PITR (binlog replay):**

```bash
# Identify the position to recover to
mysqlbinlog --start-datetime="2025-03-15 00:00:00" \
            --stop-datetime="2025-03-15 14:35:00" \
            /var/log/mysql/binlog.000042 | mysql -u root
```

### Backup Verification

A backup that has never been tested is not a backup — it is a hope. Regularly verify that backups can actually be restored.

**Verification checklist:**

```bash
# 1. Restore to a separate instance
pg_restore -d mydb_verify /backups/mydb_20250315.dump

# 2. Run row count checks
psql -d mydb_verify -c "
  SELECT 'orders' AS tbl, count(*) FROM orders
  UNION ALL
  SELECT 'users', count(*) FROM users;
"

# 3. Run application-level integrity queries
psql -d mydb_verify -c "
  SELECT count(*) FROM orders o
  LEFT JOIN users u ON o.user_id = u.id
  WHERE u.id IS NULL;
"  -- Should return 0 (no orphaned orders)

# 4. Compare checksums with production
pg_dump --schema-only mydb | md5sum
pg_dump --schema-only mydb_verify | md5sum
```

**Automate verification** — run a nightly cron job that restores the latest backup to a scratch database and validates row counts. Alert if it fails.

### Backup Rotation Strategies

Keeping every backup forever is expensive. The **Grandfather-Father-Son (GFS)** scheme balances retention cost against recovery flexibility:

```
┌───────────────────────────────────────────────────┐
│            Grandfather-Father-Son (GFS)           │
├───────────────┬──────────────┬────────────────────┤
│   Grandfathers│    Fathers   │       Sons         │
│   (Monthly)   │   (Weekly)   │     (Daily)        │
├───────────────┼──────────────┼────────────────────┤
│  Keep 12      │  Keep 4      │  Keep 7            │
│  (1 year)     │  (1 month)   │  (1 week)          │
└───────────────┴──────────────┴────────────────────┘

Day 1 ─ Day 2 ─ Day 3 ─ ... ─ Day 7  ◀── 7 daily (Son)
Week 1 ─ Week 2 ─ Week 3 ─ Week 4     ◀── 4 weekly (Father)
Jan ─ Feb ─ Mar ─ ... ─ Dec            ◀── 12 monthly (Grandfather)

Total backups retained: 7 + 4 + 12 = 23
```

| Rotation Tier | Frequency | Retention | Storage Cost |
|---|---|---|---|
| **Son** (daily) | Every day | 7 days | Low |
| **Father** (weekly) | Every Sunday | 4 weeks | Medium |
| **Grandfather** (monthly) | 1st of month | 12 months | High (archive tier) |

---

## High Availability

Backups protect against data loss; high availability (HA) protects against downtime. As discussed in *Designing Data-Intensive Applications* (DDIA) Chapter 5, the core idea is **replication** — maintaining a standby copy that can take over when the primary fails.

### Primary-Replica Failover

```
                     Normal Operation
┌──────────┐  sync/async   ┌──────────┐
│ Primary  │──────────────▶│ Replica  │
│ (writes) │  replication   │ (standby)│
└──────────┘               └──────────┘
      │
      ▼
  Application

                     After Failover
┌──────────┐               ┌──────────┐
│  Former  │               │  New     │
│ Primary  │  (offline)    │ Primary  │
│  (down)  │               │ (writes) │
└──────────┘               └──────────┘
                                │
                                ▼
                           Application
```

**Failover steps:**

1. Detect that the primary is unavailable (health checks, heartbeats)
2. Verify the replica has caught up (minimize data loss)
3. Promote the replica to primary
4. Redirect application traffic to the new primary
5. Investigate and repair the old primary
6. Re-add the old primary as a new replica

### Automatic Failover Tools

Manual failover requires an on-call engineer to be awake, diagnose the issue, and execute the promotion. Automatic failover tools reduce this to seconds.

**Patroni (PostgreSQL):**

Patroni uses a distributed consensus store (etcd, ZooKeeper, or Consul) to elect a leader and coordinate failover.

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  etcd 1  │─────│  etcd 2  │─────│  etcd 3  │
└────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │
┌────┴─────┐     ┌────┴─────┐     ┌────┴─────┐
│ Patroni  │     │ Patroni  │     │ Patroni  │
│    +     │     │    +     │     │    +     │
│ PostgreSQL│    │ PostgreSQL│    │ PostgreSQL│
│ (leader) │     │ (replica)│     │ (replica)│
└──────────┘     └──────────┘     └──────────┘
```

```yaml
# patroni.yml — key configuration
scope: mydb-cluster
name: node1

etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1 MB — don't promote if too far behind
    postgresql:
      parameters:
        max_connections: 200
        synchronous_commit: "on"
```

**Orchestrator (MySQL):**

Orchestrator maps the replication topology and performs automated failover by identifying the best promotion candidate.

```bash
# Discover and map the topology
orchestrator -c discover -i primary-host:3306

# Manually trigger failover (or let Orchestrator auto-detect)
orchestrator -c graceful-master-takeover \
  -i primary-host:3306 -d promoted-replica:3306
```

| Feature | Patroni (PostgreSQL) | Orchestrator (MySQL) |
|---|---|---|
| Consensus backend | etcd / ZooKeeper / Consul | Raft (built-in) or MySQL backend |
| Failover time | ~10-30 seconds | ~10-30 seconds |
| Topology awareness | Single primary + replicas | Complex topologies (chain, co-primary) |
| Configuration | YAML | Web UI + CLI + API |
| Split-brain protection | Fencing via DCS lease | Fencing via anti-flap, promotion rules |

### DNS and Floating IP Failover

After promoting a replica, the application needs to connect to the new primary. Two common approaches:

**Floating IP (VIP):**

```
Before failover:               After failover:
VIP: 10.0.1.100               VIP: 10.0.1.100
      │                              │
      ▼                              ▼
┌──────────┐                   ┌──────────┐
│ Primary  │                   │  New     │
│ 10.0.1.1 │                   │ Primary  │
└──────────┘                   │ 10.0.1.2 │
                               └──────────┘
```

The VIP moves to the new primary. Application connection strings stay the same.

**DNS failover:**

```bash
# Update DNS A record to point to new primary
# TTL should be low (e.g., 5-10 seconds) for fast failover
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "db-primary.internal.example.com",
        "Type": "A",
        "TTL": 5,
        "ResourceRecords": [{"Value": "10.0.1.2"}]
      }
    }]
  }'
```

| Method | Pros | Cons |
|---|---|---|
| **Floating IP** | Instant switchover, no DNS cache issues | Same subnet only, cloud VPC limits |
| **DNS failover** | Works across data centers, simple | TTL caching delays, stale DNS resolvers |
| **Proxy layer** | Application sees single endpoint | Additional latency, another failure point |

### Split-Brain Prevention

Split-brain occurs when two nodes both believe they are the primary and accept writes independently. This causes data divergence that is extremely difficult to reconcile.

As described in *DDIA* Chapter 8, distributed systems must guard against this with **fencing**:

```
           ┌──────────────────────┐
           │  SPLIT-BRAIN DANGER  │
           └──────────────────────┘

  ┌──────────┐               ┌──────────┐
  │ Node A   │  network      │ Node B   │
  │ "I am    │  partition    │ "I am    │
  │  primary"│◀─── ✕ ───▶   │  primary"│
  └──────────┘               └──────────┘
       │                          │
  Writes from                Writes from
  Client Group 1             Client Group 2
       │                          │
       ▼                          ▼
  Divergent data!            Divergent data!
```

**Prevention techniques:**

1. **Fencing tokens** — a monotonically increasing token from the consensus store. Storage rejects writes with stale tokens.
2. **STONITH** (Shoot The Other Node In The Head) — the new primary forcibly powers off the old primary before accepting writes.
3. **Quorum-based leader election** — require a majority of nodes to agree on the leader (etcd, ZooKeeper).
4. **Watchdog timers** — Patroni can configure a Linux watchdog that reboots the node if Patroni stops heartbeating to the DCS.

```yaml
# Patroni watchdog configuration
watchdog:
  mode: required        # Patroni won't start without watchdog
  device: /dev/watchdog
  safety_margin: 5      # seconds before watchdog triggers
```

---

## Monitoring and Alerting

You cannot manage what you cannot measure. Database monitoring must cover **resource utilization**, **query performance**, **replication health**, and **error rates**.

### Key Metrics to Monitor

| Category | Metric | Why It Matters | Alert Threshold (Example) |
|---|---|---|---|
| **Connections** | Active connections / max | Pool exhaustion causes app errors | > 80% of `max_connections` |
| **Query Performance** | p95 query latency | User-facing slowness | > 500ms |
| **Query Performance** | Queries per second | Traffic baseline and anomalies | Deviation > 2σ from baseline |
| **Replication** | Replication lag (seconds or bytes) | Stale reads, data loss risk on failover | > 30 seconds |
| **Disk** | Disk usage percentage | Full disk = database crash | > 80% |
| **Disk** | Write IOPS | I/O saturation stalls all queries | Sustained > 80% of provisioned |
| **Locks** | Lock wait time | Blocked transactions pile up | Any query waiting > 30 seconds |
| **Cache** | Buffer cache hit ratio | Low ratio = excessive disk reads | < 95% (PostgreSQL) |
| **Errors** | Deadlocks per minute | Application logic issues | > 0 (investigate every deadlock) |
| **Maintenance** | Dead tuple ratio (PostgreSQL) | Indicates VACUUM is falling behind | > 10% dead tuples in any table |

### PostgreSQL: pg_stat_statements

`pg_stat_statements` is the most important PostgreSQL extension for production. It records execution statistics for every distinct query.

```sql
-- Enable the extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 10 queries by total execution time
SELECT
  queryid,
  calls,
  round(total_exec_time::numeric, 2) AS total_ms,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  round((100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0))::numeric, 2)
    AS cache_hit_pct,
  query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Queries with worst cache hit ratio (I/O-heavy)
SELECT
  queryid,
  calls,
  shared_blks_read,
  shared_blks_hit,
  round((100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0))::numeric, 2)
    AS cache_hit_pct,
  query
FROM pg_stat_statements
WHERE calls > 100
ORDER BY cache_hit_pct ASC
LIMIT 10;

-- Reset statistics periodically to track recent trends
SELECT pg_stat_statements_reset();
```

### MySQL: Slow Query Log

```sql
-- Enable slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;          -- Log queries > 1 second
SET GLOBAL log_queries_not_using_indexes = 'ON';

-- View slow query log location
SHOW VARIABLES LIKE 'slow_query_log_file';
```

```bash
# Analyze slow query log with pt-query-digest
pt-query-digest /var/log/mysql/slow.log \
  --limit 20 \
  --order-by Query_time:sum
```

The **Performance Schema** provides additional detail:

```sql
-- Top queries by total latency
SELECT
  DIGEST_TEXT,
  COUNT_STAR AS calls,
  ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_sec,
  ROUND(AVG_TIMER_WAIT / 1e12, 4) AS avg_sec
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

### Prometheus Exporters

**postgres_exporter** and **mysqld_exporter** expose database metrics in Prometheus format for collection, alerting, and dashboarding.

```yaml
# docker-compose.yml — postgres_exporter
services:
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://monitor:password@db:5432/mydb?sslmode=require"
    ports:
      - "9187:9187"

  mysqld-exporter:
    image: prom/mysqld-exporter
    environment:
      DATA_SOURCE_NAME: "monitor:password@(db:3306)/"
    ports:
      - "9104:9104"
```

Key exported metrics:

| Exporter | Metric | Description |
|---|---|---|
| postgres_exporter | `pg_stat_activity_count` | Connections by state |
| postgres_exporter | `pg_stat_replication_lag` | Replication lag in bytes |
| postgres_exporter | `pg_stat_user_tables_n_dead_tup` | Dead tuples per table |
| mysqld_exporter | `mysql_global_status_threads_connected` | Current connections |
| mysqld_exporter | `mysql_slave_status_seconds_behind_master` | Replication lag |
| mysqld_exporter | `mysql_global_status_innodb_buffer_pool_reads` | Buffer pool misses |

### Alerting Rules

```yaml
# prometheus-rules.yml
groups:
  - name: database_alerts
    rules:
      - alert: HighReplicationLag
        expr: pg_stat_replication_lag > 30
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Replication lag {{ $value }}s on {{ $labels.instance }}"

      - alert: DiskSpaceCritical
        expr: (node_filesystem_avail_bytes{mountpoint="/var/lib/postgresql"} /
               node_filesystem_size_bytes{mountpoint="/var/lib/postgresql"}) < 0.15
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Database disk < 15% free on {{ $labels.instance }}"

      - alert: TooManyConnections
        expr: pg_stat_activity_count > (pg_settings_max_connections * 0.8)
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Connection usage > 80% on {{ $labels.instance }}"

      - alert: LowCacheHitRatio
        expr: pg_stat_database_blks_hit /
              (pg_stat_database_blks_hit + pg_stat_database_blks_read) < 0.95
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Cache hit ratio below 95% on {{ $labels.instance }}"

      - alert: DeadlocksDetected
        expr: rate(pg_stat_database_deadlocks[5m]) > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Deadlocks occurring on {{ $labels.instance }}"
```

---

## Security

A database breach is among the most damaging incidents an organization can face. Security must be designed in layers — no single control is sufficient.

### Principle of Least Privilege

Every database user and application should have **only the permissions required** for its function. Never use the superuser account for application connections.

```sql
-- PostgreSQL: create role hierarchy
CREATE ROLE app_readonly;
GRANT CONNECT ON DATABASE mydb TO app_readonly;
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO app_readonly;

CREATE ROLE app_readwrite;
GRANT app_readonly TO app_readwrite;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;

-- Application service account
CREATE USER app_service WITH PASSWORD 'strong-random-password';
GRANT app_readwrite TO app_service;

-- Migration account (needs DDL but not superuser)
CREATE USER migration_service WITH PASSWORD 'another-strong-password';
GRANT app_readwrite TO migration_service;
GRANT CREATE ON SCHEMA public TO migration_service;
```

```sql
-- MySQL: similar privilege separation
CREATE USER 'app_reader'@'10.0.%' IDENTIFIED BY 'strong-password';
GRANT SELECT ON mydb.* TO 'app_reader'@'10.0.%';

CREATE USER 'app_writer'@'10.0.%' IDENTIFIED BY 'strong-password';
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'app_writer'@'10.0.%';

CREATE USER 'migration'@'10.0.%' IDENTIFIED BY 'strong-password';
GRANT ALL PRIVILEGES ON mydb.* TO 'migration'@'10.0.%';
REVOKE SUPER ON *.* FROM 'migration'@'10.0.%';
```

### Encrypted Connections (TLS)

All database connections must be encrypted in transit, especially across network boundaries.

**PostgreSQL (`postgresql.conf` + `pg_hba.conf`):**

```
# postgresql.conf
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'
ssl_ca_file = '/etc/ssl/certs/ca.crt'
ssl_min_protocol_version = 'TLSv1.2'

# pg_hba.conf — require SSL for all remote connections
hostssl all all 0.0.0.0/0 scram-sha-256
```

**MySQL (`my.cnf`):**

```ini
[mysqld]
require_secure_transport = ON
ssl-ca   = /etc/mysql/ssl/ca.pem
ssl-cert = /etc/mysql/ssl/server-cert.pem
ssl-key  = /etc/mysql/ssl/server-key.pem
tls_version = TLSv1.2,TLSv1.3
```

### Encryption at Rest

Encryption at rest protects data on disk from physical theft or unauthorized access to storage volumes.

| Approach | How It Works | Key Management |
|---|---|---|
| **Filesystem encryption** (dm-crypt/LUKS) | Entire volume encrypted | OS-level key, unlocked at boot |
| **Tablespace encryption** (TDE) | Database-level transparent encryption | Key in keyring plugin/extension |
| **Cloud-managed encryption** (AWS EBS, GCP PD) | Storage layer encryption | Cloud KMS, automatic rotation |

```sql
-- PostgreSQL: pgcrypto for column-level encryption of sensitive fields
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Encrypt
INSERT INTO users (email, ssn_encrypted)
VALUES ('user@example.com', pgp_sym_encrypt('123-45-6789', 'encryption-key'));

-- Decrypt
SELECT email, pgp_sym_decrypt(ssn_encrypted, 'encryption-key') AS ssn
FROM users
WHERE email = 'user@example.com';
```

### Audit Logging

Track who did what, when. Audit logs are essential for compliance (SOC 2, HIPAA, GDPR) and incident investigation.

**PostgreSQL — pgAudit:**

```sql
-- Install and configure pgAudit
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- postgresql.conf
-- pgaudit.log = 'write, ddl'       -- Log all writes and DDL
-- pgaudit.log_catalog = off         -- Skip system catalog queries
-- pgaudit.log_relation = on         -- Log object names
```

**MySQL — audit_log plugin:**

```sql
-- Enable audit log
INSTALL PLUGIN audit_log SONAME 'audit_log.so';
SET GLOBAL audit_log_policy = 'LOGINS';       -- Log all login events
SET GLOBAL audit_log_format = 'JSON';
```

### SQL Injection Prevention

SQL injection remains the most common database attack vector. It is **entirely preventable** with proper coding practices.

```
NEVER do this (string concatenation):
  query = "SELECT * FROM users WHERE id = " + user_input

ALWAYS do this (parameterized queries):
  query = "SELECT * FROM users WHERE id = $1"   -- with parameter binding
```

```python
# Python — parameterized query (psycopg2)
cursor.execute(
    "SELECT * FROM users WHERE email = %s AND status = %s",
    (user_email, 'active')
)

# NEVER do this:
# cursor.execute(f"SELECT * FROM users WHERE email = '{user_email}'")
```

```sql
-- Database-level defense: use SECURITY DEFINER functions
CREATE FUNCTION get_user(p_email TEXT)
RETURNS TABLE (id INT, email TEXT, name TEXT)
LANGUAGE sql
SECURITY DEFINER
AS $$
  SELECT id, email, name FROM users WHERE email = p_email;
$$;

-- Application calls the function, never touches the table directly
SELECT * FROM get_user('user@example.com');
```

### Network Segmentation

Databases should never be directly accessible from the public internet.

```
┌─────────────────────────────────────────────────────────┐
│                     Public Internet                      │
└────────────────────────┬────────────────────────────────┘
                         │
                    ┌────┴────┐
                    │  Load   │
                    │ Balancer│
                    └────┬────┘
                         │
┌────────────────────────┼──── Application Subnet ────────┐
│               ┌────────┴────────┐                       │
│               │  App Servers    │                       │
│               └────────┬────────┘                       │
└────────────────────────┼────────────────────────────────┘
                         │ port 5432 only
┌────────────────────────┼──── Database Subnet ───────────┐
│               ┌────────┴────────┐                       │
│               │  Database       │                       │
│               │  (no public IP) │                       │
│               └─────────────────┘                       │
└─────────────────────────────────────────────────────────┘
```

**Rules:**

- Database servers: **no public IP**, no SSH from internet
- Security groups: allow port 5432/3306 **only from application subnet**
- Admin access: through bastion host or VPN only
- Monitor for unauthorized connection attempts

---

## Capacity Planning

Running out of disk space at 3 AM is not a capacity plan. Proactive capacity planning uses historical trends to predict future needs and provision resources before they become bottlenecks.

### Disk Growth Forecasting

```sql
-- PostgreSQL: track database size over time
SELECT
  datname,
  pg_size_pretty(pg_database_size(datname)) AS current_size,
  pg_database_size(datname) AS size_bytes
FROM pg_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY size_bytes DESC;

-- Track table growth (run weekly, store results)
SELECT
  schemaname || '.' || relname AS table_name,
  pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
  pg_size_pretty(pg_relation_size(relid)) AS table_size,
  pg_size_pretty(pg_indexes_size(relid)) AS index_size,
  n_live_tup AS live_rows
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;
```

**Forecasting formula:**

```
Daily growth rate = (size_today - size_30_days_ago) / 30
Days until 80% full = (disk_capacity * 0.8 - size_today) / daily_growth_rate
```

Set an alert when **days until 80% < 30 days** — giving you a month to provision more storage.

### Connection Limits

Connection limits are a common source of outages. Each connection consumes memory on the server.

```
Per-connection memory ≈ work_mem + temp_buffers + overhead
PostgreSQL: ~5-10 MB per connection
MySQL: ~10-12 MB per connection (with sort/join buffers)

max_connections = 200  →  ~2 GB of connection overhead
max_connections = 1000 →  ~10 GB of connection overhead
```

**Sizing rule:** Prefer fewer direct connections with a connection pooler. See [08-CONNECTION-MANAGEMENT.md](08-CONNECTION-MANAGEMENT.md) for detailed pooling strategies.

```sql
-- PostgreSQL: check current connection usage
SELECT
  count(*) AS total,
  count(*) FILTER (WHERE state = 'active') AS active,
  count(*) FILTER (WHERE state = 'idle') AS idle,
  count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_tx,
  current_setting('max_connections')::int AS max_conn
FROM pg_stat_activity;
```

### Memory Sizing

| Parameter | Database | Rule of Thumb | Notes |
|---|---|---|---|
| `shared_buffers` | PostgreSQL | 25% of system RAM | Diminishing returns above 8 GB on some workloads |
| `effective_cache_size` | PostgreSQL | 50-75% of system RAM | Hint for planner, not an allocation |
| `work_mem` | PostgreSQL | 2-8 MB | Per-sort/hash operation; careful — multiplied by parallel workers |
| `maintenance_work_mem` | PostgreSQL | 512 MB - 2 GB | Used by VACUUM, CREATE INDEX |
| `innodb_buffer_pool_size` | MySQL | 70-80% of system RAM | The single most important MySQL setting |
| `innodb_log_file_size` | MySQL | 1-2 GB per file | Larger = fewer checkpoints, better write throughput |

```
# PostgreSQL memory layout (example: 64 GB server)
shared_buffers       = 16GB     # 25% of 64 GB
effective_cache_size = 48GB     # 75% of 64 GB
work_mem             = 4MB      # conservative — 200 connections × 4 MB = 800 MB peak
maintenance_work_mem = 2GB      # for VACUUM and reindex
```

```ini
# MySQL memory layout (example: 64 GB server)
[mysqld]
innodb_buffer_pool_size = 48G       # 75% of 64 GB
innodb_buffer_pool_instances = 8    # reduce mutex contention
innodb_log_file_size = 2G
innodb_log_buffer_size = 64M
join_buffer_size = 4M
sort_buffer_size = 4M
```

### IOPS Planning

IOPS (Input/Output Operations Per Second) is often the invisible bottleneck, especially on cloud storage.

| Workload Type | Typical IOPS Need | Storage Recommendation |
|---|---|---|
| OLTP (transactions) | 3,000 - 20,000 IOPS | SSD / io2 (AWS) / pd-ssd (GCP) |
| OLAP (analytics) | Throughput-sensitive (MB/s) | Large SSD / st1 for sequential |
| Mixed | 5,000 - 30,000 IOPS | Provisioned IOPS SSD |

```sql
-- PostgreSQL: monitor I/O wait events
SELECT
  backend_type,
  wait_event_type,
  wait_event,
  count(*)
FROM pg_stat_activity
WHERE wait_event_type = 'IO'
GROUP BY 1, 2, 3
ORDER BY count(*) DESC;
```

---

## Maintenance Operations

Databases require ongoing maintenance to prevent gradual degradation. Without it, tables bloat, statistics become stale, and queries slow down over weeks and months.

### PostgreSQL: VACUUM and ANALYZE

PostgreSQL's MVCC model means deleted or updated rows leave behind **dead tuples** that consume disk space and slow scans. VACUUM reclaims this space.

```sql
-- Check tables with the most dead tuples
SELECT
  schemaname || '.' || relname AS table_name,
  n_live_tup,
  n_dead_tup,
  CASE WHEN n_live_tup > 0
    THEN round(100.0 * n_dead_tup / n_live_tup, 2)
    ELSE 0
  END AS dead_pct,
  last_vacuum,
  last_autovacuum,
  last_analyze,
  last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

**Autovacuum tuning** — the defaults are conservative. High-write tables need more aggressive settings:

```sql
-- Per-table autovacuum tuning for a high-churn table
ALTER TABLE events SET (
  autovacuum_vacuum_scale_factor = 0.01,       -- trigger at 1% dead (default 20%)
  autovacuum_vacuum_cost_delay = 2,            -- less sleep between I/O (default 2ms)
  autovacuum_analyze_scale_factor = 0.005      -- re-analyze more often
);

-- Increase autovacuum workers globally (postgresql.conf)
-- autovacuum_max_workers = 6      (default is 3)
-- autovacuum_naptime = 15s        (default 1min — check tables more often)
```

**ANALYZE** updates the planner's statistics about data distribution. Stale statistics → bad query plans.

```sql
-- Manually analyze a specific table
ANALYZE VERBOSE orders;

-- Analyze the entire database
ANALYZE;
```

### MySQL: OPTIMIZE TABLE

MySQL InnoDB does not have PostgreSQL's dead-tuple problem, but fragmentation accumulates as rows are inserted, updated, and deleted.

```sql
-- Check table fragmentation
SELECT
  TABLE_SCHEMA,
  TABLE_NAME,
  DATA_LENGTH,
  DATA_FREE,
  ROUND(100 * DATA_FREE / (DATA_LENGTH + DATA_FREE), 2) AS frag_pct
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb'
  AND DATA_FREE > 0
ORDER BY DATA_FREE DESC;

-- Defragment a table (takes a table lock — schedule during maintenance window)
OPTIMIZE TABLE mydb.orders;

-- For large tables, use ALTER TABLE ... FORCE instead (online DDL)
ALTER TABLE mydb.orders FORCE;
```

### Index Maintenance

Indexes degrade over time — B-tree pages become sparse after deletions, and indexes for removed columns linger.

```sql
-- PostgreSQL: find unused indexes
SELECT
  schemaname || '.' || relname AS table_name,
  indexrelname AS index_name,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
  idx_scan AS times_used
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'    -- keep primary keys
ORDER BY pg_relation_size(indexrelid) DESC;

-- PostgreSQL: reindex to reclaim bloated index space
REINDEX INDEX CONCURRENTLY idx_orders_created_at;

-- MySQL: find unused indexes (requires Performance Schema)
SELECT
  object_schema,
  object_name,
  index_name,
  count_star AS rows_read
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
  AND count_star = 0
  AND object_schema = 'mydb'
ORDER BY object_name;
```

See [04-QUERY-OPTIMIZATION.md](04-QUERY-OPTIMIZATION.md) for index design strategies.

### Bloat Management

Table and index bloat is a PostgreSQL-specific concern that can cause a 10 GB table to consume 50 GB on disk.

```sql
-- Estimate table bloat using pgstattuple
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT
  table_len,
  tuple_count,
  tuple_len,
  dead_tuple_count,
  dead_tuple_len,
  free_space,
  round(100.0 * free_space / table_len, 2) AS free_pct
FROM pgstattuple('orders');
```

**Bloat remediation options:**

| Method | Downtime | Notes |
|---|---|---|
| `VACUUM FULL` | ✅ Locks table | Rewrites entire table; use only as last resort |
| `pg_repack` | ❌ Minimal | Rewrites table online without heavy locks |
| `CLUSTER` | ✅ Locks table | Reorders table by index; rarely needed |
| Preventive autovacuum tuning | N/A | Best approach — prevent bloat from accumulating |

```bash
# pg_repack — online table compaction without heavy locks
pg_repack -d mydb -t orders --no-superuser-check
```

---

## Schema Management

Database schema changes are among the riskiest operations in production. A bad migration can lock tables, break application code, or corrupt data.

### Migration Tools

Use a dedicated migration tool — never apply DDL changes manually in production. See [06-MIGRATIONS.md](06-MIGRATIONS.md) for a detailed comparison of migration frameworks and safe migration patterns.

**Core principles:**

- Every schema change is versioned and checked into source control
- Migrations are idempotent or forward-only
- Rollback scripts are written and tested alongside each migration
- Migrations run through the same CI/CD pipeline as application code

### DDL Change Review Process

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────┐
│ Developer│────▶│  PR with     │────▶│  DBA / Peer  │────▶│  Staging │
│ writes   │     │  migration   │     │   review     │     │  deploy  │
│ migration│     │  + rollback  │     │              │     │  & test  │
└──────────┘     └──────────────┘     └──────────────┘     └────┬─────┘
                                                                │
                                                           ┌────┴─────┐
                                                           │Production│
                                                           │  deploy  │
                                                           └──────────┘
```

**DDL review checklist:**

| Check | Question |
|---|---|
| Lock duration | Does this DDL take an `ACCESS EXCLUSIVE` lock? For how long? |
| Table size | How large is the affected table? Is online DDL feasible? |
| Index creation | Is `CREATE INDEX CONCURRENTLY` used (PostgreSQL)? |
| NOT NULL addition | Does the column have a DEFAULT? (avoids full table rewrite in PG 11+) |
| Column type change | Does this require a table rewrite? |
| Foreign key | Does adding a FK lock the referenced table? |
| Backward compatibility | Can the old application code work with the new schema? |
| Rollback plan | Is there a tested rollback migration? |

### Staging Environment Testing

Every migration must be tested against a **production-like staging environment** before deployment to production.

**Staging requirements:**

- Same database engine and major version as production
- Representative data volume (full copy or scaled-down subset)
- Same schema, same indexes, same extensions
- Run the migration and measure: lock duration, execution time, replication lag impact

```bash
# Time the migration on staging
time psql -d staging_mydb -f migrations/20250315_add_status_column.sql

# Check for unexpected locks during migration
psql -d staging_mydb -c "
  SELECT pid, mode, granted, relation::regclass
  FROM pg_locks
  WHERE NOT granted;
"
```

---

## Performance Tuning Checklist

Performance tuning is iterative. Start with the highest-impact, lowest-effort changes and measure the result before moving to the next.

### Connection Pooling

The first performance fix for most applications is adding connection pooling. Opening a new database connection for every request adds 5-20 ms of latency and wastes server resources.

See [08-CONNECTION-MANAGEMENT.md](08-CONNECTION-MANAGEMENT.md) for detailed pooling configuration.

**Quick wins:**

| Action | Impact |
|---|---|
| Add PgBouncer or ProxySQL | Reduce connection overhead by 90%+ |
| Set pool size to `(2 × CPU cores) + disk spindles` | Prevent connection saturation |
| Set `idle_in_transaction_session_timeout` | Reclaim leaked connections |
| Enable `statement_timeout` | Kill runaway queries |

### Index Strategy

Indexes are the single biggest lever for query performance. A missing index turns a 5 ms query into a 5 second sequential scan.

See [04-QUERY-OPTIMIZATION.md](04-QUERY-OPTIMIZATION.md) for index types and design patterns.

**Quick wins:**

```sql
-- PostgreSQL: find sequential scans on large tables
SELECT
  schemaname || '.' || relname AS table_name,
  seq_scan,
  seq_tup_read,
  idx_scan,
  pg_size_pretty(pg_relation_size(relid)) AS table_size
FROM pg_stat_user_tables
WHERE seq_scan > 100
  AND pg_relation_size(relid) > 10 * 1024 * 1024   -- > 10 MB
ORDER BY seq_tup_read DESC
LIMIT 10;
```

### Query Review

Bad queries are the most common cause of database performance problems. Review the top queries from `pg_stat_statements` or the slow query log weekly.

**Query review checklist:**

| Check | Remediation |
|---|---|
| Full table scans on large tables | Add appropriate index |
| `SELECT *` in application queries | Select only needed columns |
| N+1 query patterns | Use JOINs or batch loading |
| Missing `LIMIT` on user-facing queries | Add LIMIT to prevent unbounded result sets |
| Implicit type casts in WHERE clauses | Fix types to allow index usage |
| Large `IN` lists (>1000 values) | Use temp table or `= ANY(ARRAY[...])` |
| Expensive `ORDER BY` without index | Add covering index for sort |

### Configuration Tuning

Default database configurations are optimized for compatibility, not performance. Tune these settings based on your hardware and workload.

**PostgreSQL key settings:**

```
# postgresql.conf — production tuning (64 GB RAM, SSD)
shared_buffers = 16GB
effective_cache_size = 48GB
work_mem = 8MB
maintenance_work_mem = 2GB
wal_buffers = 64MB
checkpoint_completion_target = 0.9
random_page_cost = 1.1                  # SSD — nearly same as seq_page_cost
effective_io_concurrency = 200          # SSD parallel I/O
max_parallel_workers_per_gather = 4
default_statistics_target = 200         # more accurate planner estimates
```

**MySQL key settings:**

```ini
# my.cnf — production tuning (64 GB RAM, SSD)
[mysqld]
innodb_buffer_pool_size = 48G
innodb_buffer_pool_instances = 8
innodb_log_file_size = 2G
innodb_flush_log_at_trx_commit = 1      # ACID compliance (do not change)
innodb_flush_method = O_DIRECT
innodb_io_capacity = 2000               # SSD baseline IOPS
innodb_io_capacity_max = 4000
innodb_read_io_threads = 8
innodb_write_io_threads = 8
```

---

## Disaster Recovery

High availability handles single-node failures within a region. Disaster recovery (DR) handles the loss of an entire site — data center failure, region outage, or catastrophic corruption.

As *Designing Data-Intensive Applications* (DDIA) Chapter 5 emphasizes, replication across geographically distributed data centers is the foundation of DR — but replication alone is not a DR plan.

### DR Site Setup

```
┌─────────────────────┐              ┌─────────────────────┐
│   Primary Region    │              │     DR Region       │
│   (us-east-1)       │              │   (eu-west-1)       │
│                     │  async       │                     │
│  ┌──────────┐       │  replication │  ┌──────────┐       │
│  │ Primary  │───────┼─────────────▶│  │ DR       │       │
│  │ Database │       │              │  │ Standby  │       │
│  └──────────┘       │              │  └──────────┘       │
│  ┌──────────┐       │              │  ┌──────────┐       │
│  │ Replica  │       │              │  │ Replica  │       │
│  └──────────┘       │              │  └──────────┘       │
│  ┌──────────┐       │              │                     │
│  │ Replica  │       │              │  ┌──────────┐       │
│  └──────────┘       │              │  │  Backup  │       │
│                     │              │  │  Storage │       │
│                     │              │  └──────────┘       │
└─────────────────────┘              └─────────────────────┘
```

**DR tiers:**

| Tier | Strategy | RTO | RPO | Cost |
|---|---|---|---|---|
| **Tier 1** | Active-active multi-region | ~0 (no failover needed) | ~0 | Very high |
| **Tier 2** | Warm standby with async replication | Minutes | Seconds to minutes | High |
| **Tier 3** | Pilot light (DR infra provisioned, data replicated) | Hours | Minutes | Medium |
| **Tier 4** | Backup and restore from offsite backups | Hours to days | Hours (last backup) | Low |

### Cross-Region Replication

```sql
-- PostgreSQL: streaming replication to DR site
-- On the primary (postgresql.conf):
wal_level = replica
max_wal_senders = 10
wal_keep_size = 2048          -- MB of WAL to retain

-- On the DR standby (postgresql.conf):
primary_conninfo = 'host=primary.us-east-1.internal port=5432
  user=replicator password=secret sslmode=require'
restore_command = 'aws s3 cp s3://wal-archive/%f %p'
```

```sql
-- MySQL: cross-region GTID-based replication
-- On the DR replica:
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = 'primary.us-east-1.internal',
  SOURCE_PORT = 3306,
  SOURCE_USER = 'replicator',
  SOURCE_PASSWORD = 'secret',
  SOURCE_AUTO_POSITION = 1,
  SOURCE_SSL = 1;

START REPLICA;
SHOW REPLICA STATUS\G
```

**Monitor cross-region lag closely** — network latency between regions adds to replication lag. Set alerts for lag exceeding your RPO.

### Regular DR Drills

A DR plan that has never been tested is a hope, not a plan. Schedule DR drills at least **quarterly**.

**DR drill procedure:**

1. **Announce the drill** — notify all teams, schedule a maintenance window
2. **Simulate the failure** — stop replication from primary, or isolate the primary region
3. **Execute the failover** — promote the DR standby to primary
4. **Validate the DR site** — run application smoke tests, verify data integrity
5. **Measure the results** — record actual RTO and RPO achieved
6. **Fail back** — restore the original primary as the new replica, re-establish replication
7. **Retrospective** — document what went well, what failed, and action items

| Drill Metric | Target | Actual (Record Each Drill) |
|---|---|---|
| Time to detect failure | < 1 minute | ______ |
| Time to promote DR standby | < 5 minutes | ______ |
| Time to redirect traffic | < 5 minutes | ______ |
| Total RTO achieved | < 15 minutes | ______ |
| Data loss (RPO) measured | < 1 minute | ______ |
| Smoke tests passed | 100% | ______ |

### Runbook Documentation

Every operational procedure should have a written runbook — a step-by-step document that anyone on the team can follow under pressure at 3 AM.

**Runbook template:**

```
# Runbook: Database Failover

## When to Use
- Primary database is unresponsive for > 60 seconds
- Replication lag is increasing unboundedly
- Primary host has hardware failure

## Pre-Conditions
- [ ] Confirm primary is truly down (not a network blip)
- [ ] Confirm replica lag before failover
- [ ] Notify #incident Slack channel

## Steps
1. Verify replica health:
   SELECT pg_is_in_recovery();      -- should return true
   SELECT pg_last_wal_replay_lsn(); -- note the position

2. Promote replica:
   pg_ctl promote -D /var/lib/postgresql/data

3. Verify promotion:
   SELECT pg_is_in_recovery();      -- should now return false

4. Update DNS / VIP:
   [Link to DNS update procedure]

5. Verify application connectivity:
   [Link to smoke test script]

## Rollback
- If promotion fails, fall back to restoring from backup
- [Link to backup restore runbook]

## Contacts
- DBA on-call: [PagerDuty rotation link]
- Infrastructure: [Team contact]
```

---

## Production Readiness Checklist

Use this checklist before launching a new database to production, or audit an existing one. Every item maps to a section in this document.

**Backup and Recovery:**

- [ ] Automated backups running on schedule (daily logical + continuous WAL/binlog)
- [ ] Backup verification automated (nightly restore + row count validation)
- [ ] PITR tested and documented (can restore to arbitrary point in time)
- [ ] Backup rotation policy defined (GFS or equivalent)
- [ ] Backups stored offsite (different region or cloud account)
- [ ] RTO and RPO defined and communicated to stakeholders
- [ ] Backup restore procedure documented in a runbook

**High Availability:**

- [ ] At least one synchronous or near-synchronous replica
- [ ] Automatic failover configured (Patroni / Orchestrator or managed service)
- [ ] Failover tested and documented
- [ ] Split-brain prevention mechanism in place (fencing, quorum)
- [ ] DNS/VIP failover configured with low TTL

**Monitoring and Alerting:**

- [ ] Connection count monitoring with alert at 80% of max
- [ ] Replication lag monitoring with alert threshold
- [ ] Disk usage monitoring with alert at 80% and 90%
- [ ] Query latency monitoring (p95, p99)
- [ ] `pg_stat_statements` or Performance Schema enabled
- [ ] Deadlock and lock-wait alerting configured
- [ ] Cache hit ratio monitoring
- [ ] Dashboard with key metrics visible to on-call

**Security:**

- [ ] Application uses dedicated database user (not superuser)
- [ ] Principle of least privilege applied to all accounts
- [ ] TLS required for all connections
- [ ] Encryption at rest enabled
- [ ] Database not accessible from public internet
- [ ] Network security groups restrict access to application subnet
- [ ] Audit logging enabled for DDL and privilege changes
- [ ] Application uses parameterized queries (no string concatenation)

**Capacity Planning:**

- [ ] Disk growth forecasting in place (alert when <30 days to 80%)
- [ ] Connection limits sized appropriately with pooling
- [ ] Memory parameters tuned for hardware (`shared_buffers`, `innodb_buffer_pool_size`)
- [ ] IOPS provisioned for workload (measured, not guessed)

**Maintenance:**

- [ ] Autovacuum tuned for high-write tables (PostgreSQL)
- [ ] Table fragmentation monitored (MySQL)
- [ ] Unused indexes identified and removed quarterly
- [ ] Statistics updated regularly (`ANALYZE` / histogram updates)
- [ ] Bloat monitoring in place (PostgreSQL)

**Schema Management:**

- [ ] All schema changes managed through migration tool ([06-MIGRATIONS.md](06-MIGRATIONS.md))
- [ ] DDL changes peer-reviewed before deployment
- [ ] Migrations tested on staging with production-like data
- [ ] Rollback scripts written and tested for every migration

**Performance:**

- [ ] Connection pooling in place ([08-CONNECTION-MANAGEMENT.md](08-CONNECTION-MANAGEMENT.md))
- [ ] Query review process established (weekly review of top queries)
- [ ] Key tables have appropriate indexes ([04-QUERY-OPTIMIZATION.md](04-QUERY-OPTIMIZATION.md))
- [ ] Database configuration tuned for hardware and workload
- [ ] Slow query logging enabled with review process

**Disaster Recovery:**

- [ ] DR site configured with cross-region replication
- [ ] DR failover procedure documented in runbook
- [ ] DR drill conducted within last quarter
- [ ] Actual RTO/RPO measured and meets targets
- [ ] Backups stored in DR region

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial best practices for production documentation |
