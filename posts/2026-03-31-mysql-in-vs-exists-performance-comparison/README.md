# MySQL IN vs EXISTS: Performance Comparison

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Query Optimization, Subquery

Description: Compare MySQL IN and EXISTS subquery operators on performance, execution strategy, and when each produces faster results for large and small datasets.

---

MySQL's `IN` and `EXISTS` operators both filter rows based on a subquery, but they use different execution strategies. The performance difference can be significant depending on table sizes, indexes, and MySQL's optimizer version.

## Basic Syntax

```sql
-- IN with subquery
SELECT * FROM orders
WHERE customer_id IN (
  SELECT id FROM customers WHERE country = 'US'
);

-- EXISTS with correlated subquery
SELECT * FROM orders o
WHERE EXISTS (
  SELECT 1 FROM customers c
  WHERE c.id = o.customer_id AND c.country = 'US'
);
```

Both return the same result set. The difference is in how MySQL executes them.

## How IN Works

`IN` with a subquery evaluates the subquery first and builds a result set (or a hash set in MySQL 8.0). The outer query then checks whether each row's value is in that set.

```sql
EXPLAIN SELECT * FROM orders
WHERE customer_id IN (
  SELECT id FROM customers WHERE country = 'US'
);
-- MySQL 8.0 may show: hash join, semijoin, or materialization
```

## How EXISTS Works

`EXISTS` evaluates the correlated subquery once per row of the outer query. It stops as soon as one match is found (short-circuit evaluation), which is efficient when matches are common.

```sql
EXPLAIN SELECT * FROM orders o
WHERE EXISTS (
  SELECT 1 FROM customers c
  WHERE c.id = o.customer_id AND c.country = 'US'
);
```

## MySQL's Optimizer Rewrites

Modern MySQL (5.6+, especially 8.0) often rewrites `IN` subqueries into semijoins internally. This means the raw SQL form matters less - the optimizer may produce the same execution plan for both.

```sql
-- Check optimizer trace to see rewrite decisions
SET optimizer_trace = 'enabled=on';
SELECT * FROM orders WHERE customer_id IN (SELECT id FROM customers WHERE country = 'US');
SELECT * FROM information_schema.optimizer_trace LIMIT 1\G
SET optimizer_trace = 'enabled=off';
```

## Performance Rules of Thumb

When the subquery result set is large, `IN` with good indexing or materialization is typically faster because it avoids per-row correlation.

When the outer table is large but the subquery returns very few rows, `EXISTS` can be faster because it short-circuits early.

```sql
-- Large customers table, small subset: EXISTS may win
SELECT * FROM orders o
WHERE EXISTS (
  SELECT 1 FROM customers c
  WHERE c.id = o.customer_id AND c.vip = 1
);
-- If only 10 VIP customers exist, EXISTS stops early per row

-- Many matching customers: IN or semijoin is efficient
SELECT * FROM orders
WHERE customer_id IN (
  SELECT id FROM customers WHERE country = 'US'
);
```

## NOT IN vs NOT EXISTS

This is where the difference is most critical. `NOT IN` returns incorrect results when the subquery returns any NULL values.

```sql
-- NOT IN fails silently when NULLs exist
SELECT * FROM orders
WHERE customer_id NOT IN (SELECT id FROM customers WHERE deactivated = 1);
-- If any customer.id is NULL, this returns ZERO rows (incorrect behavior)

-- NOT EXISTS handles NULLs correctly
SELECT * FROM orders o
WHERE NOT EXISTS (
  SELECT 1 FROM customers c
  WHERE c.id = o.customer_id AND c.deactivated = 1
);
```

Always use `NOT EXISTS` (or `LEFT JOIN ... WHERE ... IS NULL`) instead of `NOT IN` with a subquery to avoid NULL-related bugs.

## Summary

In MySQL 8.0, the optimizer frequently rewrites both forms to the same execution plan, so the difference is often negligible. Use `IN` when the subquery is straightforward and returns many distinct values. Use `EXISTS` when you need early termination, when the subquery is correlated and selective, or when you need `NOT EXISTS` to correctly handle NULLs. Always avoid `NOT IN` with subqueries that could return NULL values.
