# How to Find the Most Resource-Intensive Queries with sys Schema in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, sys Schema, Query, Optimization, Performance

Description: Use MySQL sys schema views to quickly identify the most CPU-intensive, I/O-heavy, and time-consuming queries affecting your database performance.

---

## Overview

The MySQL `sys` schema provides pre-built views built on top of Performance Schema that make it easy to find resource-intensive queries without writing complex SQL. The `statement_analysis` view is the primary tool for this, but several other views offer complementary perspectives.

## Using the statement_analysis View

The `statement_analysis` view shows normalized query digests with aggregated performance metrics:

```sql
SELECT
  query,
  exec_count,
  total_latency,
  avg_latency,
  max_latency,
  rows_examined_avg,
  rows_sent_avg,
  tmp_tables,
  full_scans
FROM sys.statement_analysis
ORDER BY total_latency DESC
LIMIT 10;
```

## Finding Queries with the Most Full Table Scans

```sql
SELECT
  query,
  exec_count,
  total_latency,
  full_scans
FROM sys.statement_analysis
WHERE full_scans > 0
ORDER BY full_scans DESC
LIMIT 10;
```

## Identifying High Row Examination Queries

Queries that examine far more rows than they return often lack proper indexes:

```sql
SELECT
  query,
  exec_count,
  rows_examined_avg,
  rows_sent_avg,
  ROUND(rows_examined_avg / rows_sent_avg, 0) AS exam_ratio
FROM sys.statement_analysis
WHERE rows_sent_avg > 0
  AND rows_examined_avg > 1000
ORDER BY exam_ratio DESC
LIMIT 10;
```

## Finding Queries Causing Temp Tables on Disk

Queries that spill temp tables to disk indicate `tmp_table_size` or query design issues:

```sql
SELECT
  query,
  exec_count,
  total_latency,
  tmp_disk_tables
FROM sys.statement_analysis
WHERE tmp_disk_tables > 0
ORDER BY tmp_disk_tables DESC
LIMIT 10;
```

## Using statements_with_errors_or_warnings

```sql
SELECT
  query,
  exec_count,
  errors,
  warnings,
  last_seen
FROM sys.statements_with_errors_or_warnings
ORDER BY errors DESC
LIMIT 10;
```

## Using statements_with_full_table_scans

```sql
SELECT
  query,
  exec_count,
  total_latency,
  no_index_used_count,
  no_good_index_used_count
FROM sys.statements_with_full_table_scans
ORDER BY total_latency DESC
LIMIT 10;
```

## Checking High-Frequency Queries

Sometimes low-latency queries cause problems through sheer volume:

```sql
SELECT
  query,
  exec_count,
  avg_latency,
  total_latency
FROM sys.statement_analysis
ORDER BY exec_count DESC
LIMIT 10;
```

## Summary

The `sys.statement_analysis` view and companion views like `statements_with_full_table_scans` and `statements_with_errors_or_warnings` provide fast, actionable query performance data. By sorting on total latency, full scan count, and row examination ratios, you can rapidly prioritize which queries to optimize with index additions, query rewrites, or schema changes.
