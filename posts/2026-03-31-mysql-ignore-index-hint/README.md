# How to Use IGNORE INDEX Hint in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Index, Query Optimization, Performance, Hint

Description: Learn how to use the IGNORE INDEX hint in MySQL to prevent the optimizer from using specific indexes during query execution.

---

## Overview

MySQL's `IGNORE INDEX` hint excludes one or more indexes from the optimizer's consideration. This is the opposite of `USE INDEX` - instead of suggesting which indexes to consider, you tell MySQL which ones to skip. It is useful when a specific index is causing poor query performance.

## Basic IGNORE INDEX Syntax

```sql
SELECT * FROM orders IGNORE INDEX (idx_status)
WHERE status = 'pending'
ORDER BY created_at DESC;
```

MySQL will not use `idx_status` when executing this query, forcing the optimizer to consider other indexes or a full table scan.

## Ignoring Multiple Indexes

You can exclude multiple indexes at once:

```sql
SELECT * FROM orders IGNORE INDEX (idx_status, idx_customer_id)
WHERE customer_id = 42 AND status = 'pending';
```

## Ignoring the Primary Key

You can also exclude the primary key from consideration:

```sql
SELECT * FROM orders IGNORE INDEX (PRIMARY)
WHERE id BETWEEN 1000 AND 2000;
```

## IGNORE INDEX FOR Specific Operations

Like other index hints, `IGNORE INDEX` can be scoped to specific operations:

```sql
-- Exclude index from JOIN operations
SELECT * FROM orders IGNORE INDEX FOR JOIN (idx_customer_id)
JOIN customers ON orders.customer_id = customers.id;

-- Exclude index from ORDER BY operations
SELECT * FROM orders IGNORE INDEX FOR ORDER BY (idx_created_at)
ORDER BY created_at DESC;

-- Exclude index from GROUP BY operations
SELECT status, COUNT(*) FROM orders IGNORE INDEX FOR GROUP BY (idx_status)
GROUP BY status;
```

## When to Use IGNORE INDEX

Common use cases include:

- An index has poor selectivity and the optimizer incorrectly prefers it over a more selective one
- An index causes the optimizer to choose a filesort instead of a faster execution path
- You are benchmarking query performance without a specific index before deciding to drop it
- Stale statistics are making a bad index appear attractive to the optimizer

## Practical Example

Suppose an orders table has both `idx_status` and `idx_created_at`, and the optimizer keeps picking `idx_status` for a query that mostly filters on date:

```sql
-- Check current plan
EXPLAIN SELECT * FROM orders
WHERE status = 'completed' AND created_at > '2024-01-01'
ORDER BY created_at DESC;

-- Force optimizer to ignore the poorly chosen index
EXPLAIN SELECT * FROM orders IGNORE INDEX (idx_status)
WHERE status = 'completed' AND created_at > '2024-01-01'
ORDER BY created_at DESC;
```

Compare the estimated rows and actual cost between the two plans.

## Comparing Index Hint Options

| Hint | Effect |
|------|--------|
| `USE INDEX (x)` | Only consider index x; full scan allowed |
| `FORCE INDEX (x)` | Use index x; full scan only if x is unusable |
| `IGNORE INDEX (x)` | Skip index x; consider everything else |

## Long-Term Fix

`IGNORE INDEX` should be a temporary workaround. The proper fix is usually:

```sql
-- Update statistics
ANALYZE TABLE orders;

-- Check index cardinality
SHOW INDEX FROM orders;

-- Consider dropping the index if it provides no benefit
ALTER TABLE orders DROP INDEX idx_status;
```

## Summary

`IGNORE INDEX` prevents MySQL from using specific indexes during query execution. It is useful for testing and for working around cases where the optimizer picks a suboptimal index. Use `EXPLAIN` to compare plans and treat it as a temporary measure while addressing the underlying statistics or schema issues.
