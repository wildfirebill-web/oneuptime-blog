# How to Use EXPLAIN for INSERT Statements in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, EXPLAIN, Performance

Description: Learn how to use EXPLAIN with INSERT statements in MySQL to analyze query plans and optimize data insertion performance.

---

## Using EXPLAIN with INSERT in MySQL

The `EXPLAIN` statement in MySQL is most commonly used with `SELECT` queries, but you can also use it with `INSERT`, `UPDATE`, and `DELETE` statements. When used with `INSERT ... SELECT`, `EXPLAIN` reveals how MySQL plans to read the source data - the same way it would for a regular `SELECT`. This helps you identify full table scans, missing indexes, and suboptimal join orders in the source query.

## Basic Syntax

```sql
EXPLAIN INSERT INTO target_table (col1, col2)
SELECT col1, col2
FROM source_table
WHERE condition;
```

MySQL will not actually perform the insert - it only returns the execution plan for the `SELECT` portion.

## Example: Analyzing an INSERT SELECT

Consider a scenario where you are copying filtered rows from one table to another:

```sql
EXPLAIN
INSERT INTO archived_orders (id, customer_id, total, created_at)
SELECT id, customer_id, total, created_at
FROM orders
WHERE created_at < '2024-01-01'
  AND status = 'completed';
```

The output shows the access type, key used, rows estimated, and any extra details like `Using where` or `Using filesort`.

```text
+----+-------------+--------+------+---------------+------------+---------+------+-------+-------------+
| id | select_type | table  | type | possible_keys | key        | key_len | ref  | rows  | Extra       |
+----+-------------+--------+------+---------------+------------+---------+------+-------+-------------+
|  1 | INSERT      | orders | ref  | idx_status    | idx_status | 1       | const| 12400 | Using where |
+----+-------------+--------+------+---------------+------------+---------+------+-------+-------------+
```

## Interpreting the Output

The key columns to examine are:

- `type`: Should ideally be `ref`, `range`, or `eq_ref`. Avoid `ALL` (full table scan).
- `key`: The index MySQL chose to use. `NULL` means no index is being used.
- `rows`: Estimated number of rows examined. Smaller is better.
- `Extra`: Watch for `Using filesort` or `Using temporary` which signal expensive operations.

## Using EXPLAIN ANALYZE with INSERT SELECT

MySQL 8.0.18+ supports `EXPLAIN ANALYZE`, which actually executes the query and provides real timing data. However, note that `EXPLAIN ANALYZE INSERT` will execute the insert for real:

```sql
EXPLAIN ANALYZE
INSERT INTO log_summary (user_id, event_count)
SELECT user_id, COUNT(*) AS event_count
FROM event_log
WHERE event_date = CURDATE()
GROUP BY user_id;
```

Use this carefully in production - consider wrapping it in a transaction that you roll back:

```sql
START TRANSACTION;
EXPLAIN ANALYZE
INSERT INTO log_summary (user_id, event_count)
SELECT user_id, COUNT(*) FROM event_log GROUP BY user_id;
ROLLBACK;
```

## Optimizing INSERT SELECT Performance

After reviewing the `EXPLAIN` output, common optimizations include:

```sql
-- Add a composite index to speed up the source query
ALTER TABLE orders ADD INDEX idx_status_date (status, created_at);

-- Verify the new plan
EXPLAIN
INSERT INTO archived_orders (id, customer_id, total, created_at)
SELECT id, customer_id, total, created_at
FROM orders
WHERE created_at < '2024-01-01' AND status = 'completed';
```

## Checking INSERT ... VALUES

For plain `INSERT ... VALUES`, `EXPLAIN` returns minimal output because there is no source query to analyze. The useful diagnostics come from `INSERT ... SELECT` or `INSERT ... TABLE` forms.

```sql
-- This produces trivial output - not very useful
EXPLAIN INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
```

## Summary

Using `EXPLAIN` with `INSERT ... SELECT` in MySQL helps you identify inefficiencies in the source query before running large data migrations or batch inserts. Focus on avoiding full table scans, ensuring proper indexes, and eliminating expensive sort and temporary table operations in the source `SELECT` to make your inserts faster and more efficient.
