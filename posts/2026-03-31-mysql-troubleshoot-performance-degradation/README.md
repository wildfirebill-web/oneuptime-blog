# How to Troubleshoot MySQL Performance Degradation Over Time

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance, Index, InnoDB, Troubleshooting

Description: Learn how to diagnose MySQL performance degradation over time caused by table bloat, stale statistics, index fragmentation, and growing query plans.

---

## Why Performance Degrades Over Time

MySQL performance often degrades gradually rather than suddenly. Common causes include table fragmentation after large deletes, stale optimizer statistics, index bloat, a growing slow query log, and increasing data volume that pushes working sets beyond the buffer pool.

## Step 1: Check for Slow Queries

Enable and review the slow query log to identify newly slow queries:

```sql
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;
```

Analyze with `pt-query-digest` or the built-in `mysqldumpslow`:

```bash
mysqldumpslow -s t -t 20 /var/log/mysql/slow.log
```

## Step 2: Check Table Fragmentation

High `data_free` values indicate fragmented tables:

```sql
SELECT
  table_schema,
  table_name,
  ROUND(data_length  / 1024 / 1024, 1) AS data_mb,
  ROUND(index_length / 1024 / 1024, 1) AS index_mb,
  ROUND(data_free    / 1024 / 1024, 1) AS free_mb
FROM information_schema.TABLES
WHERE data_free > 100 * 1024 * 1024  -- More than 100MB free
ORDER BY data_free DESC
LIMIT 10;
```

Reclaim space and improve sequential scan performance:

```sql
OPTIMIZE TABLE fragmented_table;
```

## Step 3: Update Optimizer Statistics

Stale statistics cause the optimizer to choose bad execution plans:

```sql
ANALYZE TABLE orders;
ANALYZE TABLE users;
```

Or update all tables in a database:

```bash
mysqlcheck -u root -p --analyze myapp_db
```

After analysis, re-run `EXPLAIN` on affected queries to verify the optimizer now picks correct indexes.

## Step 4: Check Index Efficiency

Index cardinality decreases relative to table growth over time:

```sql
SELECT
  table_name,
  index_name,
  cardinality,
  seq_in_index,
  column_name
FROM information_schema.STATISTICS
WHERE table_schema = 'myapp'
ORDER BY table_name, index_name, seq_in_index;
```

Use `EXPLAIN` on slow queries to check whether indexes are being used:

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 42 AND status = 'pending'\G
```

Look for `type = ALL` (full scan) or `Extra = Using filesort` as red flags.

## Step 5: Check InnoDB Buffer Pool Hit Rate

As data grows beyond buffer pool capacity, hit rates drop and disk I/O increases:

```sql
SELECT
  (1 - (
    variable_value / (
      SELECT variable_value FROM performance_schema.global_status
      WHERE variable_name = 'Innodb_buffer_pool_read_requests'
    )
  )) * 100 AS hit_rate_pct
FROM performance_schema.global_status
WHERE variable_name = 'Innodb_buffer_pool_reads';
```

A hit rate below 95% indicates the buffer pool is too small for the working set.

## Step 6: Growing Number of Rows Changing Query Plans

Queries that were fast on small tables may switch to full scans as tables grow:

```sql
-- Check row counts
SELECT table_name, table_rows
FROM information_schema.TABLES
WHERE table_schema = 'myapp'
ORDER BY table_rows DESC
LIMIT 20;
```

Add composite indexes for frequently filtered columns as data grows:

```sql
CREATE INDEX idx_orders_user_status_date
  ON orders (user_id, status, created_at);
```

## Step 7: Review Long-Running Transactions

Old transactions hold undo log space and prevent InnoDB from purging old row versions, causing MVCC overhead to grow:

```sql
SELECT trx_id, trx_started, trx_state,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS seconds
FROM information_schema.INNODB_TRX
ORDER BY trx_started
LIMIT 10;
```

Kill idle transactions that have been open for hours.

## Summary

MySQL performance degradation over time is most commonly caused by table fragmentation, stale optimizer statistics, growing data volume outpacing the buffer pool, and index cardinality becoming less selective. Run `ANALYZE TABLE` and `OPTIMIZE TABLE` regularly, monitor buffer pool hit rates, and review `EXPLAIN` output on slow queries after significant data growth to catch plan regressions early.
