# How to Find Slow Queries Using Performance Schema in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance Schema, Slow Query, Query Optimization, Database

Description: Use MySQL Performance Schema to identify slow queries, top resource consumers, and execution patterns without enabling the slow query log.

---

The slow query log is the traditional tool for finding slow queries in MySQL, but the Performance Schema offers a richer, queryable approach. You can find top queries by execution time, I/O, or error count - all without writing to disk or parsing log files.

## Prerequisites

Ensure Performance Schema is enabled and statement consumers are active:

```sql
SHOW VARIABLES LIKE 'performance_schema';
```

If it shows `OFF`, enable it in `my.cnf`:

```bash
[mysqld]
performance_schema=ON
```

Enable the required consumers:

```sql
UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME IN (
  'statements_digest',
  'events_statements_current',
  'events_statements_history',
  'events_statements_history_long'
);
```

Enable statement instruments:

```sql
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'statement/%';
```

## Finding Top Queries by Total Execution Time

The `events_statements_summary_by_digest` table aggregates statistics by normalized query pattern:

```sql
SELECT
  SCHEMA_NAME,
  DIGEST_TEXT,
  COUNT_STAR AS exec_count,
  ROUND(SUM_TIMER_WAIT / 1e12, 3) AS total_time_sec,
  ROUND(AVG_TIMER_WAIT / 1e12, 6) AS avg_time_sec,
  ROUND(MAX_TIMER_WAIT / 1e12, 6) AS max_time_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE SCHEMA_NAME IS NOT NULL
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

## Finding Queries with the Highest Average Latency

```sql
SELECT
  DIGEST_TEXT,
  COUNT_STAR AS exec_count,
  ROUND(AVG_TIMER_WAIT / 1e12, 6) AS avg_time_sec,
  ROUND(MAX_TIMER_WAIT / 1e12, 6) AS max_time_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE COUNT_STAR > 5
  AND SCHEMA_NAME IS NOT NULL
ORDER BY AVG_TIMER_WAIT DESC
LIMIT 10;
```

## Finding Queries with High Row Examination

Queries that examine many rows but return few are strong candidates for index optimization:

```sql
SELECT
  DIGEST_TEXT,
  COUNT_STAR,
  SUM_ROWS_EXAMINED,
  SUM_ROWS_SENT,
  ROUND(SUM_ROWS_EXAMINED / NULLIF(SUM_ROWS_SENT, 0), 1) AS rows_examined_per_sent,
  ROUND(AVG_TIMER_WAIT / 1e12, 6) AS avg_time_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_ROWS_SENT > 0
  AND SCHEMA_NAME IS NOT NULL
ORDER BY rows_examined_per_sent DESC
LIMIT 10;
```

## Finding Queries That Cause Temporary Tables

Queries using temporary tables on disk are often bottlenecks:

```sql
SELECT
  DIGEST_TEXT,
  COUNT_STAR,
  SUM_CREATED_TMP_DISK_TABLES AS disk_tmp_tables,
  SUM_CREATED_TMP_TABLES AS mem_tmp_tables,
  ROUND(AVG_TIMER_WAIT / 1e12, 6) AS avg_time_sec
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_CREATED_TMP_DISK_TABLES > 0
ORDER BY SUM_CREATED_TMP_DISK_TABLES DESC
LIMIT 10;
```

## Viewing Currently Running Slow Queries

To see what is actively running and how long it has been executing:

```sql
SELECT
  t.PROCESSLIST_ID,
  t.PROCESSLIST_USER,
  t.PROCESSLIST_DB,
  t.PROCESSLIST_TIME AS running_sec,
  s.SQL_TEXT
FROM performance_schema.threads t
JOIN performance_schema.events_statements_current s
  ON t.THREAD_ID = s.THREAD_ID
WHERE t.PROCESSLIST_TIME > 5
  AND t.PROCESSLIST_COMMAND = 'Query'
ORDER BY t.PROCESSLIST_TIME DESC;
```

## Resetting Statistics

After tuning, reset the digest statistics to get a clean baseline:

```sql
TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;
```

## Summary

MySQL Performance Schema provides powerful query analysis through the `events_statements_summary_by_digest` table, which aggregates normalized query statistics across execution count, total time, row examination, and temporary table usage. Enable statement instruments and consumers, then query the digest table to find slow queries by total time, average latency, or scan efficiency. Unlike the slow query log, this approach is fully queryable in SQL and does not require log file parsing.
