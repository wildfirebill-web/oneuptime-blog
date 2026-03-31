# How to Use USE INDEX Hint in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Index, Query Optimization, Performance, Hint

Description: Learn how to use the USE INDEX hint in MySQL to suggest which indexes the optimizer should consider when executing a query.

---

## Overview

MySQL's optimizer selects indexes automatically. The `USE INDEX` hint narrows down the list of indexes the optimizer may consider, excluding all others. Unlike `FORCE INDEX`, the optimizer can still choose a full table scan if it estimates that is cheaper than using the suggested index.

## Basic USE INDEX Syntax

```sql
SELECT * FROM orders USE INDEX (idx_customer_id)
WHERE customer_id = 42;
```

The hint is placed between the table name and the WHERE clause. MySQL will only consider `idx_customer_id` and won't evaluate other indexes on the table.

## Using Multiple Index Hints

You can suggest multiple indexes:

```sql
SELECT * FROM orders USE INDEX (idx_customer_id, idx_status)
WHERE customer_id = 42 AND status = 'pending';
```

The optimizer will consider only these two indexes and choose the best one.

## Ignoring All Indexes

Passing an empty list tells MySQL to use no indexes at all (equivalent to a full table scan):

```sql
SELECT * FROM orders USE INDEX ()
WHERE customer_id = 42;
```

This is useful for benchmarking - comparing query performance with and without indexes.

## USE INDEX FOR Specific Operations

You can scope the hint to specific query operations:

```sql
-- For the JOIN operation
SELECT * FROM orders USE INDEX FOR JOIN (idx_customer_id)
JOIN customers ON orders.customer_id = customers.id;

-- For ORDER BY
SELECT * FROM orders USE INDEX FOR ORDER BY (idx_created_at)
ORDER BY created_at DESC;

-- For GROUP BY
SELECT customer_id, COUNT(*) FROM orders USE INDEX FOR GROUP BY (idx_customer_id)
GROUP BY customer_id;
```

## Difference Between USE INDEX and FORCE INDEX

| Hint | Behavior |
|------|----------|
| `USE INDEX` | Optimizer considers only the listed indexes; full scan still allowed |
| `FORCE INDEX` | Optimizer uses the index; full scan only if index unusable |
| `IGNORE INDEX` | Optimizer excludes the listed indexes from consideration |

## Verifying the Hint with EXPLAIN

Always confirm that the hint is working as expected:

```sql
EXPLAIN SELECT * FROM orders USE INDEX (idx_customer_id)
WHERE customer_id = 42;
```

Check the `possible_keys` and `key` columns. If `key` is NULL, the optimizer chose a full scan despite the hint - this is expected behavior with `USE INDEX` when the optimizer finds the full scan cheaper.

If you need to guarantee the index is used, switch to `FORCE INDEX`.

## Practical Example: Fixing a Bad Query Plan

Suppose EXPLAIN shows `type: ALL` (full scan) on a large table when an index should be used:

```sql
-- Before hint
EXPLAIN SELECT * FROM events WHERE event_date = '2024-06-01';

-- Add hint to guide the optimizer
EXPLAIN SELECT * FROM events USE INDEX (idx_event_date)
WHERE event_date = '2024-06-01';
```

If statistics are stale, run ANALYZE TABLE first:

```sql
ANALYZE TABLE events;
```

## Summary

`USE INDEX` is a soft hint that restricts which indexes MySQL considers during query planning. The optimizer can still fall back to a full table scan if it estimates that is cheaper. Use it to guide the optimizer when index selection is suboptimal, and combine it with `EXPLAIN` to verify the actual execution plan.
