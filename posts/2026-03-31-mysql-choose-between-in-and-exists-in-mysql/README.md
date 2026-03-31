# How to Choose Between IN and EXISTS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, In Operator, Exists, Subquery, Performance

Description: Learn when to use IN vs EXISTS in MySQL subqueries, how they handle NULLs differently, and which performs better in different scenarios.

---

## Introduction

Both `IN` and `EXISTS` can filter rows based on whether a matching value exists in another table, but they work differently internally and have important differences in NULL handling, performance, and readability. Choosing the right one can significantly affect query correctness and speed.

## How IN Works

`IN` evaluates the subquery, builds the complete result set, and then checks each outer row against that set.

```sql
SELECT id, name FROM customers
WHERE id IN (SELECT customer_id FROM orders WHERE status = 'completed');
```

MySQL evaluates the subquery first, producing a list like `[1, 5, 8, 12, ...]`, then checks if each customer's `id` is in that list.

## How EXISTS Works

`EXISTS` runs the subquery for each row of the outer query and stops as soon as one matching row is found (short-circuit evaluation).

```sql
SELECT id, name FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders o
  WHERE o.customer_id = c.id AND o.status = 'completed'
);
```

For each customer, MySQL searches the orders table and returns as soon as a match is found.

## NULL Handling Difference

This is the most important practical difference. `IN` returns UNKNOWN (not FALSE) when any value in the list is NULL, which can cause no rows to be returned unexpectedly.

```sql
-- If orders has any row with customer_id = NULL:
-- NOT IN returns zero rows (unexpected)
SELECT id FROM customers WHERE id NOT IN (SELECT customer_id FROM orders);

-- NOT EXISTS is NULL-safe - always works correctly
SELECT id FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

**Rule: Always use `NOT EXISTS` instead of `NOT IN` when NULLs may be present in the subquery column.**

## Performance Comparison

### When IN is Better

- Subquery returns a small, fixed list.
- The subquery is not correlated (does not reference outer columns).
- The subquery result can be cached and reused.

```sql
-- IN works well here: subquery returns a small set
SELECT name FROM products
WHERE category_id IN (SELECT id FROM categories WHERE type = 'digital');
```

### When EXISTS is Better

- The outer table is small and the subquery table is large.
- The subquery is correlated.
- You only need to know IF a match exists, not WHAT the match is.
- The subquery column may contain NULLs.

```sql
-- EXISTS is better when outer set is small and inner set is large
SELECT c.name FROM customers c
WHERE EXISTS (
  SELECT 1 FROM large_activity_log l WHERE l.customer_id = c.id
);
```

## Modern MySQL Optimization

MySQL 8.0+ is quite good at transforming `IN` with subqueries into efficient `semijoin` operations, which are similar to how `EXISTS` works internally. In many cases, the difference in performance is minimal.

```sql
EXPLAIN
SELECT id FROM customers WHERE id IN (SELECT customer_id FROM orders);
-- Look for "Using semi-join" or similar in Extra column
```

## Readability Comparison

`IN` is more readable for simple equality checks:

```sql
-- Clean and readable
WHERE department IN ('Engineering', 'Marketing', 'Finance')

-- Subquery with IN
WHERE id IN (SELECT id FROM priority_customers)
```

`EXISTS` is better when the subquery has complex conditions:

```sql
-- More expressive with EXISTS
WHERE EXISTS (
  SELECT 1 FROM orders o
  JOIN order_items oi ON o.id = oi.order_id
  WHERE o.customer_id = c.id
    AND oi.product_id = 42
    AND o.status = 'completed'
)
```

## Side-by-Side Comparison

```sql
-- Using IN
SELECT p.id, p.name
FROM products p
WHERE p.category_id IN (
  SELECT id FROM categories WHERE active = 1
);

-- Using EXISTS
SELECT p.id, p.name
FROM products p
WHERE EXISTS (
  SELECT 1 FROM categories c
  WHERE c.id = p.category_id AND c.active = 1
);

-- Using JOIN (often fastest)
SELECT DISTINCT p.id, p.name
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE c.active = 1;
```

## Decision Guide

| Situation | Recommended |
|-----------|-------------|
| Simple value list | IN |
| NOT IN with possible NULLs | NOT EXISTS |
| Checking existence only | EXISTS |
| Correlated subquery | EXISTS |
| Large subquery result | EXISTS |
| Small, known value set | IN |
| Maximum performance | JOIN |

## Summary

Use `IN` for simple equality checks against small result sets. Use `EXISTS` when checking for the presence of related rows, especially with correlated subqueries or when NULLs may be involved. Always use `NOT EXISTS` instead of `NOT IN` when the subquery might return NULL values. For the best performance on large datasets, consider rewriting as a JOIN and verifying with EXPLAIN.
