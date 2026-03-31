# How to Use the events_statements_summary Tables in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance Schema, Statement, Query, Diagnostic

Description: Learn how to use MySQL Performance Schema events_statements_summary tables to identify slow queries, high-frequency statements, and execution patterns.

---

## Overview

The `events_statements_summary_*` family of tables in Performance Schema aggregates SQL statement execution statistics. They are invaluable for identifying the queries that consume the most time, cause the most errors, or execute most frequently.

## Key Summary Tables

MySQL provides several statement summary tables:

- `events_statements_summary_by_digest` - aggregated by SQL digest (normalized query)
- `events_statements_summary_by_thread_by_event_name` - per thread and event name
- `events_statements_summary_by_account_by_event_name` - per account
- `events_statements_summary_by_user_by_event_name` - per user
- `events_statements_summary_by_host_by_event_name` - per host
- `events_statements_summary_global_by_event_name` - global totals

## Enabling Statement Instrumentation

```sql
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'statement/%';

UPDATE performance_schema.setup_consumers
SET ENABLED = 'YES'
WHERE NAME LIKE '%statements%';
```

## Finding the Slowest Queries by Average Latency

```sql
SELECT
  DIGEST_TEXT,
  COUNT_STAR AS executions,
  ROUND(AVG_TIMER_WAIT / 1e9, 2) AS avg_ms,
  ROUND(MAX_TIMER_WAIT / 1e9, 2) AS max_ms,
  SUM_ROWS_EXAMINED,
  SUM_ROWS_SENT
FROM performance_schema.events_statements_summary_by_digest
ORDER BY AVG_TIMER_WAIT DESC
LIMIT 10;
```

## Finding Queries with the Most Total Execution Time

```sql
SELECT
  SCHEMA_NAME,
  DIGEST_TEXT,
  COUNT_STAR,
  ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_sec,
  ROUND(AVG_TIMER_WAIT / 1e9, 2) AS avg_ms
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

## Identifying Queries with High Row Examination Ratios

A high ratio of rows examined to rows sent indicates missing indexes:

```sql
SELECT
  DIGEST_TEXT,
  COUNT_STAR,
  SUM_ROWS_EXAMINED,
  SUM_ROWS_SENT,
  ROUND(SUM_ROWS_EXAMINED / SUM_ROWS_SENT, 0) AS examination_ratio
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_ROWS_SENT > 0
ORDER BY examination_ratio DESC
LIMIT 10;
```

## Queries Causing the Most Errors

```sql
SELECT
  DIGEST_TEXT,
  COUNT_STAR,
  SUM_ERRORS,
  SUM_WARNINGS,
  ROUND(SUM_ERRORS / COUNT_STAR * 100, 1) AS error_pct
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_ERRORS > 0
ORDER BY SUM_ERRORS DESC
LIMIT 10;
```

## Per-User Statement Statistics

```sql
SELECT
  USER,
  EVENT_NAME,
  COUNT_STAR,
  ROUND(SUM_TIMER_WAIT / 1e12, 2) AS total_sec
FROM performance_schema.events_statements_summary_by_user_by_event_name
WHERE COUNT_STAR > 0
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;
```

## Resetting Statement Statistics

```sql
TRUNCATE TABLE performance_schema.events_statements_summary_by_digest;
```

## Summary

The `events_statements_summary_by_digest` table is one of the most powerful tools for MySQL query analysis. By examining average latency, total execution time, rows examined ratios, and error counts, you can systematically identify and resolve query performance problems without needing slow query log access.
