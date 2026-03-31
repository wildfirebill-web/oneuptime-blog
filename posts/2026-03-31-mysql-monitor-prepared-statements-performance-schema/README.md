# How to Monitor Prepared Statements with Performance Schema in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance Schema, Prepared Statement, Monitoring, Optimization

Description: Learn how to monitor prepared statement usage, execution counts, and latency using MySQL Performance Schema to improve application query efficiency.

---

## Overview

Prepared statements reduce parse overhead by allowing MySQL to compile a query once and execute it many times with different parameters. Performance Schema tracks prepared statement lifecycle events, giving you insight into how your application uses them and whether they are working as intended.

## Key Performance Schema Tables for Prepared Statements

The primary table is `prepared_statements_instances`, which shows all currently allocated prepared statements:

```sql
SELECT
  STATEMENT_ID,
  STATEMENT_NAME,
  SQL_TEXT,
  OWNER_THREAD_ID,
  OWNER_OBJECT_TYPE,
  COUNT_REPREPARE,
  COUNT_EXECUTE,
  SUM_TIMER_EXECUTE / 1e9 AS total_exec_ms,
  AVG_TIMER_EXECUTE / 1e9 AS avg_exec_ms
FROM performance_schema.prepared_statements_instances
ORDER BY SUM_TIMER_EXECUTE DESC
LIMIT 20;
```

## Detecting Re-prepare Events

Re-prepares happen when a table structure changes or metadata is invalidated. Frequent re-prepares indicate schema churn or unstable query plans:

```sql
SELECT
  SQL_TEXT,
  COUNT_REPREPARE,
  COUNT_EXECUTE,
  ROUND(COUNT_REPREPARE / COUNT_EXECUTE * 100, 1) AS reprepare_pct
FROM performance_schema.prepared_statements_instances
WHERE COUNT_REPREPARE > 0
ORDER BY COUNT_REPREPARE DESC;
```

## Finding Slow Prepared Statements

```sql
SELECT
  SQL_TEXT,
  COUNT_EXECUTE,
  ROUND(AVG_TIMER_EXECUTE / 1e9, 2) AS avg_exec_ms,
  ROUND(MAX_TIMER_EXECUTE / 1e9, 2) AS max_exec_ms,
  SUM_ROWS_EXAMINED,
  SUM_ROWS_SENT
FROM performance_schema.prepared_statements_instances
WHERE COUNT_EXECUTE > 0
ORDER BY AVG_TIMER_EXECUTE DESC
LIMIT 10;
```

## Checking Row Examination Efficiency

```sql
SELECT
  SQL_TEXT,
  COUNT_EXECUTE,
  SUM_ROWS_EXAMINED,
  SUM_ROWS_SENT,
  CASE WHEN SUM_ROWS_SENT > 0
    THEN ROUND(SUM_ROWS_EXAMINED / SUM_ROWS_SENT, 0)
    ELSE NULL
  END AS exam_ratio
FROM performance_schema.prepared_statements_instances
WHERE COUNT_EXECUTE > 0
ORDER BY exam_ratio DESC
LIMIT 10;
```

## Checking Max Prepared Statements Limit

```sql
SHOW VARIABLES LIKE 'max_prepared_stmt_count';
SELECT COUNT(*) FROM performance_schema.prepared_statements_instances;
```

If the count approaches `max_prepared_stmt_count`, increase it or investigate connection leaks where applications are not deallocating prepared statements.

## Monitoring Statement Events for Prepared Execution

You can also track prepared statement executions through `events_statements_history_long`:

```sql
SELECT
  SQL_TEXT,
  TIMER_WAIT / 1e9 AS exec_ms,
  ROWS_EXAMINED,
  ROWS_SENT
FROM performance_schema.events_statements_history_long
WHERE SQL_TEXT IS NOT NULL
  AND ROWS_EXAMINED > 10000
ORDER BY TIMER_WAIT DESC
LIMIT 10;
```

## Summary

The `prepared_statements_instances` table in Performance Schema is the most direct way to monitor prepared statement usage in MySQL. Track `COUNT_REPREPARE`, average execution time, and row examination ratios to ensure your application is getting the full performance benefit of prepared statements without unexpected re-parses or inefficient query plans.
