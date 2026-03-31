# How to Enable Performance Schema in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance Schema, Monitoring, Performance Tuning, Database Diagnostics

Description: Learn how to enable and configure MySQL Performance Schema to monitor query execution, memory usage, and server activity with practical examples.

---

## What Is Performance Schema?

The MySQL **Performance Schema** is a feature that instruments server internals so you can monitor query execution, wait events, memory usage, and more. It stores data in in-memory tables in the `performance_schema` database and has minimal overhead when properly configured.

It is enabled by default in MySQL 5.6.6 and later.

## Checking If Performance Schema Is Enabled

```sql
SHOW VARIABLES LIKE 'performance_schema';
```

Output:

```text
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| performance_schema | ON    |
+--------------------+-------+
```

## Enabling Performance Schema

### Via my.cnf (Permanent)

Add the following to your MySQL configuration file:

```text
[mysqld]
performance_schema = ON
```

Then restart MySQL:

```bash
sudo systemctl restart mysql
```

### Verifying After Restart

```sql
SELECT * FROM performance_schema.setup_instruments LIMIT 5;
```

## Understanding Setup Tables

Performance Schema uses setup tables to control what is monitored:

| Table | Purpose |
|-------|---------|
| `setup_instruments` | Controls which events are tracked |
| `setup_consumers` | Controls where event data is stored |
| `setup_actors` | Controls which users are monitored |
| `setup_objects` | Controls which objects are monitored |
| `setup_threads` | Controls which threads are monitored |

## Enabling All Instruments

By default, not all instruments are enabled. To enable everything:

```sql
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES';

UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES';
```

## Enabling Specific Instrument Categories

Enable only statement instruments:

```sql
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'statement/%';
```

Enable only wait instruments:

```sql
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'wait/%';
```

## Viewing Top Slow Queries

```sql
SELECT
  DIGEST_TEXT,
  COUNT_STAR AS exec_count,
  ROUND(AVG_TIMER_WAIT / 1e9, 2) AS avg_ms,
  ROUND(SUM_TIMER_WAIT / 1e9, 2) AS total_ms
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

## Viewing Memory Usage by Component

```sql
SELECT
  EVENT_NAME,
  CURRENT_NUMBER_OF_BYTES_USED / 1024 / 1024 AS current_mb
FROM performance_schema.memory_summary_global_by_event_name
ORDER BY CURRENT_NUMBER_OF_BYTES_USED DESC
LIMIT 10;
```

## Viewing Wait Events

```sql
SELECT
  EVENT_NAME,
  COUNT_STAR,
  ROUND(SUM_TIMER_WAIT / 1e12, 3) AS total_seconds
FROM performance_schema.events_waits_summary_global_by_event_name
WHERE COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

## Viewing Per-Thread Activity

```sql
SELECT
  t.PROCESSLIST_USER,
  t.PROCESSLIST_HOST,
  t.PROCESSLIST_DB,
  s.COUNT_STAR,
  ROUND(s.SUM_TIMER_WAIT / 1e9, 2) AS total_ms
FROM performance_schema.events_statements_summary_by_thread_by_event_name s
JOIN performance_schema.threads t ON s.THREAD_ID = t.THREAD_ID
WHERE t.PROCESSLIST_USER IS NOT NULL
ORDER BY s.SUM_TIMER_WAIT DESC
LIMIT 10;
```

## Resetting Performance Schema Data

To clear accumulated data without restarting:

```sql
TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;
TRUNCATE TABLE performance_schema.events_waits_summary_global_by_event_name;
```

## Disabling Performance Schema

To disable Performance Schema (not recommended in production without a reason):

```text
[mysqld]
performance_schema = OFF
```

## Performance Overhead

Performance Schema is designed to have minimal overhead. The overhead typically ranges from 1-5% on most workloads. You can reduce it by enabling only the instruments you need:

```sql
-- Disable all first, then selectively enable
UPDATE performance_schema.setup_instruments SET ENABLED = 'NO', TIMED = 'NO';
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'statement/sql/%';
```

## Summary

MySQL Performance Schema is a powerful diagnostics tool that tracks server internals including queries, waits, memory, and threads. It is enabled by default in modern MySQL versions and can be configured to monitor specific areas with minimal overhead. Use the setup tables to control what is instrumented and the summary tables to analyze performance bottlenecks.
