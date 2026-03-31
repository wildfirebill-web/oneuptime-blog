# How to Track MySQL Handler Statistics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Handler Statistics, Performance, Index Usage, Monitoring

Description: Learn how to read and interpret MySQL Handler statistics to understand index usage, full table scan rates, and storage engine row access patterns.

---

## What Are Handler Statistics?

Handler statistics are MySQL server-level counters that track how the storage engine interacts with data at the row level. They reveal whether queries are using indexes efficiently, how many rows are read vs. written, and whether full table scans are occurring. Unlike `EXPLAIN`, handler stats show cumulative behavior across all queries since the server started.

## Viewing Handler Statistics

```sql
-- Show all Handler counters
SHOW GLOBAL STATUS LIKE 'Handler%';
```

```text
+----------------------------+----------+
| Variable_name              | Value    |
+----------------------------+----------+
| Handler_commit             | 45230    |
| Handler_delete             | 1203     |
| Handler_discover           | 0        |
| Handler_external_lock      | 98210    |
| Handler_mrr_init           | 0        |
| Handler_prepare            | 12540    |
| Handler_read_first         | 8901     |
| Handler_read_key           | 4523100  |
| Handler_read_last          | 12       |
| Handler_read_next          | 2345000  |
| Handler_read_prev          | 0        |
| Handler_read_rnd           | 98000    |
| Handler_read_rnd_next      | 8900000  |
| Handler_rollback           | 5        |
| Handler_savepoint          | 0        |
| Handler_update             | 34500    |
| Handler_write              | 12340    |
+----------------------------+----------+
```

## Understanding the Key Handler Counters

### Index Read Counters

| Counter | What It Means |
|---------|---------------|
| `Handler_read_key` | Rows read by an index lookup (good - means index is being used) |
| `Handler_read_next` | Rows read by advancing to the next index entry (range scans, ORDER BY) |
| `Handler_read_prev` | Rows read by going to the previous index entry (descending index scans) |
| `Handler_read_first` | First row of an index read (full index scans starting from the beginning) |
| `Handler_read_last` | Last row of an index read (ORDER BY DESC) |

### Full Table / Random Read Counters

| Counter | What It Means |
|---------|---------------|
| `Handler_read_rnd` | Rows read in fixed position (after a sort, indicating filesort) |
| `Handler_read_rnd_next` | Rows read sequentially through a table file (full table scan) |

A high `Handler_read_rnd_next` relative to `Handler_read_key` indicates too many full table scans.

## Calculating the Index Hit Ratio

```sql
-- Query to calculate read efficiency
SELECT
  gs1.VARIABLE_VALUE AS read_key,
  gs2.VARIABLE_VALUE AS read_rnd_next,
  ROUND(
    gs1.VARIABLE_VALUE / (gs1.VARIABLE_VALUE + gs2.VARIABLE_VALUE) * 100, 2
  ) AS index_read_pct
FROM
  performance_schema.global_status gs1,
  performance_schema.global_status gs2
WHERE gs1.VARIABLE_NAME = 'Handler_read_key'
  AND gs2.VARIABLE_NAME = 'Handler_read_rnd_next';
```

A healthy OLTP system should have an index read percentage well above 90%. A low percentage means many queries are doing full table scans.

## Resetting Counters for Analysis

To measure handler activity over a specific window, flush the status counters before starting your workload:

```sql
-- Reset all status counters
FLUSH STATUS;

-- Run your workload or queries here

-- Check handler stats after the workload
SHOW SESSION STATUS LIKE 'Handler%';
```

Note that `FLUSH STATUS` resets session-level counters. For global counters, you need a server restart or use the delta approach by recording values before and after.

## Per-Session Handler Counters

Handler counters are also available at the session level, giving you read stats for only the current connection:

```sql
-- Reset session counters
FLUSH STATUS;

-- Run a specific query
SELECT * FROM orders WHERE customer_id = 42 ORDER BY order_date DESC;

-- Check what the query did
SHOW SESSION STATUS LIKE 'Handler_read%';
```

```text
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| Handler_read_first     | 0     |
| Handler_read_key       | 1     |
| Handler_read_last      | 0     |
| Handler_read_next      | 15    |
| Handler_read_prev      | 0     |
| Handler_read_rnd       | 0     |
| Handler_read_rnd_next  | 0     |
+------------------------+-------+
```

This shows the query used an index (`Handler_read_key = 1`) and read 15 consecutive index entries (`Handler_read_next = 15`) - efficient behavior for a range scan.

## Identifying Tables with High Scan Rates

Use performance_schema to find which tables have the most full scans:

```sql
SELECT
  OBJECT_SCHEMA AS db_name,
  OBJECT_NAME AS table_name,
  COUNT_READ AS total_reads,
  COUNT_FETCH AS total_fetches,
  SUM_TIMER_READ / 1e12 AS read_secs
FROM performance_schema.table_io_waits_summary_by_table
ORDER BY SUM_TIMER_READ DESC
LIMIT 20;
```

## Write Handler Counters

```sql
SHOW GLOBAL STATUS LIKE 'Handler_write';
SHOW GLOBAL STATUS LIKE 'Handler_update';
SHOW GLOBAL STATUS LIKE 'Handler_delete';
```

High `Handler_write` values indicate frequent INSERT operations. Track these over time to understand write throughput and compare with your expected workload.

## Summary

MySQL Handler statistics provide a server-wide view of how storage engines access rows, whether through indexed lookups or full scans. A high `Handler_read_rnd_next` to `Handler_read_key` ratio is a strong indicator that queries are missing index coverage. Use `FLUSH STATUS` before and after running specific workloads to isolate their handler impact, and complement Handler stats with `performance_schema.table_io_waits_summary_by_table` for per-table drill-down.
