# How to Rewrite NOT IN to LEFT JOIN IS NULL in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Query Optimization, NOT IN, LEFT JOIN, Anti-Join

Description: Learn why NOT IN performs poorly in MySQL, how NULL values cause correctness issues, and how to rewrite NOT IN to LEFT JOIN IS NULL for better performance.

---

## Overview

`NOT IN` with a subquery is one of the most common sources of performance problems and subtle bugs in MySQL. It has poor index utilization, mishandles NULL values, and often produces full table scans. Rewriting `NOT IN` as a LEFT JOIN anti-join pattern fixes both the performance and correctness issues.

## The NULL Problem with NOT IN

The most critical issue with NOT IN is how it handles NULL values:

```sql
-- Setup
CREATE TABLE customers (id INT PRIMARY KEY, name VARCHAR(100));
CREATE TABLE orders (id INT, customer_id INT);

INSERT INTO customers VALUES (1, 'Alice'), (2, 'Bob'), (3, 'Carol');
INSERT INTO orders VALUES (1, 1), (2, NULL); -- Note: NULL customer_id

-- This returns NO rows, even though customers 2 and 3 have no orders!
SELECT id, name FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);
```

When the subquery contains even one NULL, `NOT IN` returns no rows because `id NOT IN (1, NULL)` evaluates to NULL (unknown), not TRUE.

## The LEFT JOIN IS NULL Anti-join Pattern

This is the correct and performant replacement:

```sql
-- Correct rewrite that handles NULLs properly
SELECT c.id, c.name
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.customer_id IS NULL;
```

This returns customers 2 and 3, which is the intended result.

## Performance Comparison

For a large `orders` table with 1 million rows:

```sql
-- Check NOT IN performance
EXPLAIN
SELECT id FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders WHERE customer_id IS NOT NULL);

-- Check LEFT JOIN IS NULL performance
EXPLAIN
SELECT c.id
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.customer_id IS NULL;
```

The LEFT JOIN version typically shows `type: ref` for the join and uses indexes, while NOT IN may show `type: ALL` (full scan of the subquery result).

## Alternative: NOT EXISTS

`NOT EXISTS` is also correct and avoids the NULL problem:

```sql
-- NOT EXISTS is NULL-safe and often performs similarly to LEFT JOIN
SELECT c.id, c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

MySQL's optimizer often rewrites `NOT EXISTS` as an anti-join internally, making it similar in performance to the LEFT JOIN pattern.

## Real-World Example: Finding Inactive Users

```sql
-- Find users who have never placed an order
-- BAD: NOT IN with potential NULLs
SELECT id, email FROM users
WHERE id NOT IN (SELECT user_id FROM orders);

-- GOOD: LEFT JOIN anti-join
SELECT u.id, u.email
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.user_id IS NULL;

-- Alternative: NOT EXISTS
SELECT u.id, u.email
FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

## When NOT IN is Safe to Use

`NOT IN` with a static list of non-NULL literals is safe and acceptable:

```sql
-- Safe: static list with no NULLs
SELECT * FROM orders WHERE status NOT IN ('cancelled', 'refunded');

-- Safe: subquery on a NOT NULL column
SELECT id FROM customers
WHERE id NOT IN (
    SELECT customer_id FROM orders
    WHERE customer_id IS NOT NULL
);
```

## Adding the Right Index

Ensure the JOIN column is indexed to maximize the anti-join performance:

```sql
-- Index on the foreign key column
CREATE INDEX idx_orders_customer ON orders (customer_id);

-- Now the anti-join uses this index efficiently
SELECT c.id, c.name
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.customer_id IS NULL;
```

## Summary

`NOT IN` with subqueries has two problems: it mishandles NULL values (returning no rows when the subquery contains any NULL) and it often performs poorly. The LEFT JOIN IS NULL anti-join pattern solves both issues - it is NULL-safe and uses indexes efficiently. `NOT EXISTS` is another correct alternative that MySQL often optimizes to an anti-join internally. Reserve `NOT IN` for static lists of known non-NULL literals, and always use LEFT JOIN IS NULL or NOT EXISTS for subquery-based exclusion logic.
