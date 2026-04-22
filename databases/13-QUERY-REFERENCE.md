# 13. Database Query Reference

A practical copy-paste reference for operational, monitoring, and troubleshooting queries across PostgreSQL, MySQL, and SQL Server. Use this document when you need fast visibility into health, locking, query performance, storage, or engine-specific production issues.

> This guide complements:
> - [04-QUERY-OPTIMIZATION.md](04-QUERY-OPTIMIZATION.md) for plan reading and tuning concepts
> - [09-BEST-PRACTICES.md](09-BEST-PRACTICES.md) for operational recommendations
> - [10-ANTI-PATTERNS.md](10-ANTI-PATTERNS.md) for failure modes and detection patterns

## Table of Contents

1. [How to Use This Reference](#how-to-use-this-reference)
2. [PostgreSQL Queries](#postgresql-queries)
   - [Active Sessions and Long Transactions](#active-sessions-and-long-transactions)
   - [Top Queries with `pg_stat_statements`](#top-queries-with-pg_stat_statements)
   - [Replication Lag](#replication-lag)
3. [MySQL Queries](#mysql-queries)
   - [Replication and GTID Status](#replication-and-gtid-status)
   - [InnoDB Health](#innodb-health)
   - [Metadata Locks](#metadata-locks)
   - [Charset and Collation Drift](#charset-and-collation-drift)
4. [SQL Server Queries](#sql-server-queries)
   - [Top Queries via Query Store](#top-queries-via-query-store)
   - [Wait Statistics](#wait-statistics)
   - [Heaps and Forwarded Records](#heaps-and-forwarded-records)
   - [Log Growth and Azure SQL Alternatives](#log-growth-and-azure-sql-alternatives)
5. [Query Usage Checklist](#query-usage-checklist)
6. [Version History](#version-history)

---

## How to Use This Reference

These queries are designed for diagnosis, not blind automation.

- **Start with the least invasive query.** Read metadata and system views before enabling heavier diagnostics.
- **Scope every query to the database, schema, or table you care about.** Production metadata can be noisy.
- **Capture a before/after snapshot.** A single result is less useful than a trend across two or three runs.
- **Know the platform.** Some SQL Server queries work on boxed SQL Server but not Azure SQL Database; some MySQL views differ between 5.7 and 8.0.

---

## PostgreSQL Queries

### Active Sessions and Long Transactions

```sql
-- Who is connected right now?
SELECT pid,
       usename,
       application_name,
       state,
       wait_event_type,
       wait_event,
       now() - query_start AS query_age,
       LEFT(query, 160)    AS query
FROM   pg_stat_activity
WHERE  datname = current_database()
ORDER BY query_start NULLS LAST;

-- Long-running transactions: look for sessions that block VACUUM cleanup
SELECT pid,
       usename,
       now() - xact_start  AS txn_age,
       now() - query_start AS query_age,
       state,
       LEFT(query, 160)    AS query
FROM   pg_stat_activity
WHERE  xact_start IS NOT NULL
ORDER BY xact_start ASC;
```

Use these first when the database feels "stuck", autovacuum falls behind, or locks start piling up.

### Top Queries with `pg_stat_statements`

```sql
-- Requires CREATE EXTENSION pg_stat_statements;
SELECT
  queryid,
  calls,
  ROUND(total_exec_time::numeric, 2) AS total_ms,
  ROUND(mean_exec_time::numeric, 2)  AS avg_ms,
  rows,
  LEFT(query, 200)                   AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Most I/O-heavy queries
SELECT
  queryid,
  calls,
  shared_blks_read,
  shared_blks_hit,
  ROUND(
    100.0 * shared_blks_hit
    / NULLIF(shared_blks_hit + shared_blks_read, 0), 2
  ) AS cache_hit_pct,
  LEFT(query, 200) AS query
FROM pg_stat_statements
WHERE calls > 50
ORDER BY shared_blks_read DESC
LIMIT 20;
```

### Replication Lag

```sql
-- Primary: replication lag by standby
SELECT application_name,
       state,
       sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS bytes_lag
FROM   pg_stat_replication;

-- Standby: replay delay
SELECT now() - pg_last_xact_replay_timestamp() AS replay_delay;
```

If replay delay keeps growing, investigate network latency, replica CPU, storage latency, or a long-running query on the replica.

---

## MySQL Queries

### Replication and GTID Status

```sql
SHOW VARIABLES LIKE 'gtid_mode';
SHOW VARIABLES LIKE 'enforce_gtid_consistency';
SHOW VARIABLES LIKE 'binlog_format';

SHOW REPLICA STATUS\G
```

Look for:

- `gtid_mode = ON`
- `binlog_format = ROW`
- `Seconds_Behind_Source` not climbing continuously
- `Last_IO_Error` / `Last_SQL_Error` empty

### InnoDB Health

```sql
-- Buffer pool hit ratio
SELECT
  FORMAT(
    100 - (100 * (reads.VARIABLE_VALUE / reqs.VARIABLE_VALUE)), 4
  ) AS buffer_pool_hit_pct
FROM performance_schema.global_status AS reads
JOIN performance_schema.global_status AS reqs
  ON reqs.VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
WHERE reads.VARIABLE_NAME = 'Innodb_buffer_pool_reads';

-- History list length
SELECT NAME, COUNT
FROM   information_schema.INNODB_METRICS
WHERE  NAME = 'trx_rseg_history_len';

-- Long-running transactions
SELECT trx_id,
       trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS seconds_active,
       trx_rows_locked,
       trx_query
FROM   information_schema.innodb_trx
ORDER BY trx_started ASC;
```

### Metadata Locks

```sql
-- MySQL 8.0+: inspect metadata locks around DDL
SELECT
  ml.OBJECT_SCHEMA,
  ml.OBJECT_NAME,
  ml.LOCK_TYPE,
  ml.LOCK_STATUS,
  t.PROCESSLIST_ID,
  t.PROCESSLIST_INFO
FROM performance_schema.metadata_locks AS ml
JOIN performance_schema.threads        AS t
  ON t.THREAD_ID = ml.OWNER_THREAD_ID
WHERE ml.OBJECT_SCHEMA = DATABASE()
  AND ml.OBJECT_NAME   = 'orders';

-- MySQL 8.0+: row-lock waits (not metadata locks)
SELECT *
FROM performance_schema.data_lock_waits;
```

`information_schema.innodb_lock_waits` was deprecated in MySQL 5.7 and removed in MySQL 8.0. Prefer `performance_schema.data_locks`, `data_lock_waits`, and `metadata_locks`.

### Charset and Collation Drift

```sql
SELECT table_schema, table_name, table_collation
FROM   information_schema.tables
WHERE  table_schema NOT IN ('information_schema','mysql','performance_schema','sys')
  AND  table_collation NOT LIKE 'utf8mb4%'
ORDER BY table_schema, table_name;

SELECT table_schema, table_name, column_name,
       character_set_name, collation_name
FROM   information_schema.columns
WHERE  character_set_name IS NOT NULL
  AND  character_set_name <> 'utf8mb4'
  AND  table_schema NOT IN ('information_schema','mysql','performance_schema','sys')
ORDER BY table_schema, table_name, column_name;
```

---

## SQL Server Queries

### Top Queries via Query Store

```sql
SELECT TOP 20
    OBJECT_NAME(qsq.object_id)             AS object_name,
    qsq.query_id,
    qsrs.count_executions,
    ROUND(qsrs.avg_duration / 1000.0, 2)   AS avg_duration_ms,
    ROUND(qsrs.avg_cpu_time / 1000.0, 2)   AS avg_cpu_ms,
    ROUND(qsrs.avg_logical_io_reads, 0)    AS avg_logical_reads,
    LEFT(qsqt.query_sql_text, 200)         AS query_text
FROM sys.query_store_runtime_stats  AS qsrs
JOIN sys.query_store_plan           AS qsqp ON qsqp.plan_id       = qsrs.plan_id
JOIN sys.query_store_query          AS qsq  ON qsq.query_id       = qsqp.query_id
JOIN sys.query_store_query_text     AS qsqt ON qsqt.query_text_id = qsq.query_text_id
ORDER BY qsrs.avg_duration DESC;
```

Use Query Store before adding hints or indexes. It is the fastest way to prove whether a regression is real and whether it came from plan drift.

### Wait Statistics

```sql
SELECT TOP 20
    wait_type,
    waiting_tasks_count,
    ROUND(wait_time_ms / 1000.0, 1) AS wait_time_sec,
    ROUND(
      100.0 * wait_time_ms / NULLIF(SUM(wait_time_ms) OVER (), 0), 2
    ) AS pct_of_total
FROM sys.dm_os_wait_stats
WHERE wait_type NOT IN (
    'SLEEP_TASK','BROKER_TO_FLUSH','BROKER_EVENTHANDLER',
    'REQUEST_FOR_DEADLOCK_SEARCH','CHECKPOINT_QUEUE',
    'SQLTRACE_BUFFER_FLUSH','CLR_AUTO_EVENT',
    'DISPATCHER_QUEUE_SEMAPHORE','XE_TIMER_EVENT',
    'FT_IFTS_SCHEDULER_IDLE_WAIT','WAITFOR',
    'LAZYWRITER_SLEEP','LOGMGR_QUEUE',
    'ONDEMAND_TASK_QUEUE','XE_DISPATCHER_WAIT'
)
ORDER BY wait_time_ms DESC;
```

Wait stats tell you **what the server is waiting on**, which is usually more actionable than just knowing CPU or duration.

### Heaps and Forwarded Records

```sql
-- Find user tables stored as heaps
SELECT
    OBJECT_SCHEMA_NAME(i.object_id) AS schema_name,
    OBJECT_NAME(i.object_id)        AS table_name,
    SUM(p.rows)                     AS row_count
FROM sys.indexes    AS i
JOIN sys.partitions AS p
  ON p.object_id = i.object_id AND p.index_id = i.index_id
WHERE i.type = 0
  AND OBJECTPROPERTY(i.object_id, 'IsUserTable') = 1
GROUP BY i.object_id
ORDER BY row_count DESC;

-- Inspect one heap for forwarded records
SELECT
    OBJECT_NAME(ips.object_id)      AS table_name,
    ips.forwarded_fetch_count,
    ips.page_count,
    ips.avg_fragmentation_in_percent
FROM sys.dm_db_index_physical_stats(
    DB_ID(), OBJECT_ID('dbo.heap_table'), NULL, NULL, 'DETAILED'
) AS ips
WHERE ips.index_id = 0;
```

### Log Growth and Azure SQL Alternatives

```sql
-- Boxed SQL Server / SQL Server on VMs: current log sizes
SELECT
    DB_NAME(database_id) AS db_name,
    name                 AS file_name,
    size * 8 / 1024      AS size_mb,
    FILEPROPERTY(name, 'SpaceUsed') * 8 / 1024 AS used_mb,
    growth * 8 / 1024    AS growth_increment_mb,
    CASE is_percent_growth WHEN 1 THEN 'percent' ELSE 'mb' END AS growth_type
FROM sys.master_files
WHERE type_desc = 'LOG';

-- Azure SQL Database: recent resource pressure snapshot
SELECT TOP 20
    end_time,
    avg_cpu_percent,
    avg_data_io_percent,
    avg_log_write_percent,
    avg_storage_percent
FROM sys.dm_db_resource_stats
ORDER BY end_time DESC;
```

For Azure SQL Database, use:

- **Azure Monitor metrics** for storage, DTU/vCore pressure, and log write pressure
- **Extended Events** for detailed file-size / storage troubleshooting
- **Intelligent Insights** for automated performance regression hints

---

## Query Usage Checklist

Before pasting a query into production:

- [ ] I know whether the query is read-only or changes state
- [ ] I know whether the view/DMV resets on restart
- [ ] I scoped the query to the target database, schema, table, or time window
- [ ] I know whether the query is valid on PostgreSQL, MySQL 5.7, MySQL 8.0, boxed SQL Server, or Azure SQL Database
- [ ] I captured enough context (timestamp, workload state, before/after values) to interpret the result later

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026 | Initial query reference for PostgreSQL, MySQL, and SQL Server operational troubleshooting |
