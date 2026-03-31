# What Is EXPLAIN ANALYZE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, EXPLAIN ANALYZE, Query Optimization, Performance Tuning

Description: EXPLAIN ANALYZE actually executes a query and shows the real runtime statistics alongside estimated costs, making it far more accurate than plain EXPLAIN.

---

## Overview

`EXPLAIN ANALYZE` was introduced in MySQL 8.0.18. Unlike plain `EXPLAIN` which only shows the optimizer's *estimates*, `EXPLAIN ANALYZE` actually **runs the query** and returns both the estimated values and the actual runtime metrics - including actual row counts, execution time, and loop counts.

This makes it invaluable for catching cases where the optimizer's estimates are badly wrong.

## Basic Syntax

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
```

Note: `EXPLAIN ANALYZE` always uses the TREE format output.

## Example Output

```sql
EXPLAIN ANALYZE
SELECT o.id, c.name, o.amount
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'US';
```

```text
-> Nested loop inner join  (cost=152.50 rows=100) (actual time=0.234..15.432 rows=87 loops=1)
    -> Filter: (c.country = 'US')  (cost=50.20 rows=50) (actual time=0.102..5.123 rows=42 loops=1)
        -> Table scan on c  (cost=50.20 rows=500) (actual time=0.089..4.900 rows=500 loops=1)
    -> Index lookup on o using idx_customer (customer_id=c.id)  (cost=0.90 rows=2) (actual time=0.012..0.245 rows=2 loops=42)
```

## Reading EXPLAIN ANALYZE Output

Each node shows:

```text
-> <operation>  (cost=<estimated_cost> rows=<estimated_rows>)
                (actual time=<first_row_ms>..<last_row_ms> rows=<actual_rows> loops=<loop_count>)
```

Key metrics to compare:

| Metric | Description |
|--------|-------------|
| `cost` | Optimizer's estimated cost |
| `rows` (estimated) | Rows the optimizer predicted |
| `actual time` | Real wall-clock milliseconds |
| `rows` (actual) | Rows actually processed |
| `loops` | How many times this node was executed |

## Spotting Optimizer Estimate Errors

```sql
EXPLAIN ANALYZE SELECT * FROM events WHERE event_type = 'click';
```

```text
-> Filter: (events.event_type = 'click')  (cost=5000.00 rows=100) (actual time=1.2..890.4 rows=450000 loops=1)
    -> Table scan on events  (cost=5000.00 rows=1000000) (actual time=1.1..700.0 rows=1000000 loops=1)
```

The optimizer estimated 100 rows but got 450,000 - a massive underestimate. Running `ANALYZE TABLE events` to update statistics would help:

```sql
ANALYZE TABLE events;
```

## Comparing Before and After Index Addition

```sql
-- Before index
EXPLAIN ANALYZE SELECT * FROM logs WHERE user_id = 100;
-- actual time=0.5..2340.0 rows=5000 loops=1 (table scan)

-- Add index
ALTER TABLE logs ADD INDEX idx_user_id (user_id);

-- After index
EXPLAIN ANALYZE SELECT * FROM logs WHERE user_id = 100;
-- actual time=0.1..3.2 rows=5000 loops=1 (index lookup)
```

## Key Differences: EXPLAIN vs EXPLAIN ANALYZE

| Feature | EXPLAIN | EXPLAIN ANALYZE |
|---------|---------|-----------------|
| Executes query | No | Yes |
| Shows estimates | Yes | Yes |
| Shows actual rows | No | Yes |
| Shows actual time | No | Yes |
| Safe for heavy queries | Yes | Use with caution |
| Available since | Always | MySQL 8.0.18 |

## Caution with Data-Modifying Statements

`EXPLAIN ANALYZE` actually runs the statement. For DML, use with care:

```sql
-- This WILL delete rows!
EXPLAIN ANALYZE DELETE FROM temp_data WHERE created_at < '2026-01-01';
```

Wrap in a transaction to safely test:

```sql
BEGIN;
EXPLAIN ANALYZE DELETE FROM temp_data WHERE created_at < '2026-01-01';
ROLLBACK;
```

## Summary

`EXPLAIN ANALYZE` provides ground-truth query profiling by executing the query and reporting actual row counts and timing alongside optimizer estimates. It is significantly more useful than plain `EXPLAIN` for diagnosing optimizer mistakes and verifying that index changes actually improve performance. Use it routinely during query optimization but be mindful that it executes the query fully.
