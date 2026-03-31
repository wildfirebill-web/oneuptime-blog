# How to Identify Using Temporary Using EXPLAIN in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, EXPLAIN, Performance, Query, Index

Description: Learn how to detect Using temporary in MySQL EXPLAIN output, understand what triggers internal temporary tables, and optimize queries to avoid them.

---

## What Does "Using Temporary" Mean?

When MySQL shows `Using temporary` in the `Extra` column of `EXPLAIN`, it means MySQL created an internal temporary table to process your query. This happens when MySQL cannot complete an intermediate step - like deduplication, grouping, or sorting - by reading data in a single pass.

Temporary tables are stored in memory when they fit within `tmp_table_size` and `max_heap_table_size`. When they exceed these limits, they spill to disk, which is significantly slower.

## Detecting Using Temporary with EXPLAIN

```sql
EXPLAIN SELECT status, COUNT(*) FROM orders GROUP BY status;
```

```text
id | type | key  | rows   | Extra
1  | ALL  | NULL | 800000 | Using temporary; Using filesort
```

The double indicator `Using temporary; Using filesort` is very common with `GROUP BY` and `DISTINCT` on unindexed columns.

## Common Triggers

### GROUP BY without an index

```sql
-- No index on the group column
EXPLAIN SELECT department, AVG(salary) FROM employees GROUP BY department;
-- Extra: Using temporary; Using filesort
```

### DISTINCT on unindexed columns

```sql
EXPLAIN SELECT DISTINCT city FROM customers;
-- Extra: Using temporary
```

### ORDER BY with a different column than GROUP BY

```sql
EXPLAIN SELECT status, COUNT(*) FROM orders
GROUP BY status
ORDER BY COUNT(*) DESC;
-- Extra: Using temporary; Using filesort
```

### UNION without ALL

```sql
EXPLAIN
SELECT name FROM customers
UNION
SELECT name FROM prospects;
-- Extra: Using temporary
```

## Fixing Using Temporary

### Add an index for GROUP BY

```sql
CREATE INDEX idx_department ON employees(department);

EXPLAIN SELECT department, AVG(salary) FROM employees GROUP BY department;
-- Extra may now show: Using index (if covering) or no temporary
```

### Use UNION ALL when duplicates are acceptable

```sql
-- UNION ALL avoids the deduplication step (no temporary table)
EXPLAIN
SELECT name FROM customers
UNION ALL
SELECT name FROM prospects;
-- Extra: NULL (no temporary table)
```

### Increase temporary table size in memory

```sql
-- Check current limits
SHOW VARIABLES LIKE 'tmp_table_size';
SHOW VARIABLES LIKE 'max_heap_table_size';

-- Increase to keep temp tables in memory
SET SESSION tmp_table_size = 64 * 1024 * 1024;       -- 64MB
SET SESSION max_heap_table_size = 64 * 1024 * 1024;  -- 64MB
```

### Rewrite GROUP BY with covering index

```sql
-- Create a covering index that includes the GROUP BY column and aggregate column
CREATE INDEX idx_dept_salary ON employees(department, salary);

EXPLAIN SELECT department, AVG(salary) FROM employees GROUP BY department;
-- Extra: Using index (best case - no temp table, no filesort)
```

## Monitoring Temporary Table Usage

```sql
-- Check how often temp tables spill to disk
SHOW GLOBAL STATUS LIKE 'Created_tmp%';
```

```text
Created_tmp_disk_tables  | 1243   <- disk spills (bad)
Created_tmp_files        | 8
Created_tmp_tables       | 45201  <- total temp tables created
```

A high ratio of `Created_tmp_disk_tables` to `Created_tmp_tables` indicates your temp table memory limits are too low.

## Summary

`Using temporary` in EXPLAIN output means MySQL is doing extra work to process intermediate results. The most effective fixes are adding indexes that match your GROUP BY or DISTINCT columns, or rewriting UNION to UNION ALL when duplicate elimination is not needed. Monitor `Created_tmp_disk_tables` in `SHOW STATUS` to understand how often this hits disk. Keeping that ratio low is essential for maintaining query performance at scale.
