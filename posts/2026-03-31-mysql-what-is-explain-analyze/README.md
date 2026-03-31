# What Is EXPLAIN ANALYZE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, EXPLAIN ANALYZE, Query Optimization, Execution Plan, Performance

Description: EXPLAIN ANALYZE in MySQL executes a query, measures actual runtime statistics, and compares them to the optimizer's estimates to reveal query plan accuracy.

---

## Overview

`EXPLAIN ANALYZE` is a MySQL 8.0.18+ statement that both explains and actually executes the query, then shows the real runtime statistics alongside the optimizer's estimates. While `EXPLAIN` only shows what MySQL plans to do, `EXPLAIN ANALYZE` shows what MySQL actually did: the true number of rows processed, the actual time spent at each node, and how many times each operation was executed. This makes it invaluable for diagnosing cases where the optimizer's estimates are wrong.

## Basic Usage

```sql
EXPLAIN ANALYZE
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE c.status = 'active'
GROUP BY c.id;
```

**Important**: `EXPLAIN ANALYZE` actually runs the query. Do not use it on `UPDATE` or `DELETE` statements in production without testing first. For DML, use `EXPLAIN ANALYZE` inside a transaction and roll back:

```sql
BEGIN;
EXPLAIN ANALYZE DELETE FROM old_logs WHERE created_at < '2025-01-01';
ROLLBACK;
```

## Output Format

`EXPLAIN ANALYZE` always uses the TREE format. Each node shows:

```
-> Aggregate: count(o.id)  (cost=1250 rows=320) (actual time=45.2..45.2 rows=318 loops=1)
    -> Hash Join  (cost=980 rows=2700) (actual time=12.1..42.3 rows=2845 loops=1)
        -> Filter: (c.status = 'active')  (cost=420 rows=320) (actual time=0.1..8.2 rows=318 loops=1)
            -> Table scan on customers  (cost=420 rows=3200) (actual time=0.1..6.8 rows=3200 loops=1)
        -> Hash  (cost=560 rows=5500) (actual time=11.9..11.9 rows=5500 loops=1)
            -> Table scan on orders  (cost=560 rows=5500) (actual time=0.2..8.4 rows=5500 loops=1)
```

Each node displays:
- `cost=N rows=N`: optimizer's estimated cost and row count
- `actual time=start..end rows=N loops=N`: measured time (milliseconds), actual rows, and loop count

## Identifying Estimate Errors

The most valuable use of `EXPLAIN ANALYZE` is comparing estimated vs actual rows:

```
-> Filter: (status = 'pending')  (cost=10000 rows=10000) (actual time=5.2..8.1 rows=3 loops=1)
```

The optimizer estimated 10,000 rows but only 3 were returned. This indicates bad statistics -- update them:

```sql
ANALYZE TABLE orders;
```

Or increase the statistics sample size:

```sql
ALTER TABLE orders STATS_SAMPLE_PAGES = 200;
ANALYZE TABLE orders;
```

## Detecting Slow Nodes

`EXPLAIN ANALYZE` shows where time is actually spent:

```
-> Sort: total DESC  (cost=5000 rows=5000) (actual time=890.5..890.9 rows=5000 loops=1)
    -> Table scan on orders  (cost=4800 rows=5000) (actual time=0.1..200.3 rows=5000 loops=1)
```

The sort takes 890ms. Adding an index on `total` might eliminate the sort.

## Reading the loops Field

`loops=N` means that node executed N times (common in JOIN inner loops):

```
-> Index lookup on order_items using idx_order_id (order_id=o.id)
   (actual time=0.1..0.2 rows=3 loops=1250)
```

`loops=1250` means this index lookup ran 1,250 times (once per outer loop row). Multiply `actual time` by `loops` to get total time: `0.2ms * 1250 = 250ms total`.

## EXPLAIN ANALYZE vs EXPLAIN FORMAT=TREE

- `EXPLAIN FORMAT=TREE`: Shows the plan without running the query.
- `EXPLAIN ANALYZE`: Runs the query and shows actual vs estimated metrics.

Always start with `EXPLAIN FORMAT=TREE` for safe inspection, then use `EXPLAIN ANALYZE` to validate with real data.

## Summary

`EXPLAIN ANALYZE` executes the query and overlays real execution statistics on the optimizer's plan, revealing discrepancies between estimated and actual row counts, identifying which nodes consume the most time, and exposing loops in nested-loop joins. It is the most powerful MySQL query diagnostic tool available and is essential for resolving cases where `EXPLAIN` alone does not explain observed slowness. Available from MySQL 8.0.18.
