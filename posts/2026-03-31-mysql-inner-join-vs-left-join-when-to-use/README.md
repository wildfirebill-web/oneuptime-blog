# MySQL INNER JOIN vs LEFT JOIN: When to Use Each

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Join, Query

Description: Understand the difference between MySQL INNER JOIN and LEFT JOIN, when each returns different results, and how to choose the right join for your query.

---

Joins are central to relational database queries. `INNER JOIN` and `LEFT JOIN` are the two most commonly used join types, and choosing the wrong one produces incorrect results - not just slower ones.

## INNER JOIN

`INNER JOIN` returns only rows where a match exists in both tables. Non-matching rows from either table are excluded.

```sql
-- Returns only customers who have placed at least one order
SELECT c.name, o.id AS order_id, o.total
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id;
```

If a customer has no orders, they do not appear in the result. If an order has no matching customer (orphaned record), it also does not appear.

## LEFT JOIN

`LEFT JOIN` (also written `LEFT OUTER JOIN`) returns all rows from the left table, with matched rows from the right table. When there is no match, right-table columns are NULL.

```sql
-- Returns ALL customers, with NULL order columns for those with no orders
SELECT c.name, o.id AS order_id, o.total
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id;

-- Result includes:
-- Alice | 101 | 49.99
-- Bob   | NULL | NULL  (no orders)
-- Carol | 102 | 89.00
```

## Finding Non-Matching Rows

A common use of `LEFT JOIN` is finding rows with no match in the other table:

```sql
-- Find customers who have NEVER placed an order
SELECT c.id, c.name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;
```

This is the standard "anti-join" pattern. You cannot do this with an `INNER JOIN`.

## Performance Considerations

`INNER JOIN` can be faster in some cases because MySQL can filter rows earlier in the execution plan. However, with proper indexes the difference is often minimal.

```sql
-- Ensure the join column is indexed
CREATE INDEX idx_orders_customer ON orders (customer_id);

EXPLAIN SELECT c.name, o.total
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id
WHERE c.country = 'US';
```

## Common Mistakes

Using `INNER JOIN` when you need `LEFT JOIN` causes silent data loss - rows quietly disappear from results.

```sql
-- Wrong: orders with no assigned sales rep are excluded
SELECT o.id, s.name AS rep
FROM orders o
INNER JOIN sales_reps s ON s.id = o.rep_id;
-- Orders where rep_id IS NULL disappear

-- Correct: shows all orders, NULL rep_name for unassigned ones
SELECT o.id, s.name AS rep
FROM orders o
LEFT JOIN sales_reps s ON s.id = o.rep_id;
```

## Multiple Joins

You can mix join types in the same query:

```sql
-- INNER JOIN requires a customer, LEFT JOIN allows missing shipping record
SELECT o.id, c.name, s.address AS ship_to
FROM orders o
INNER JOIN customers c ON c.id = o.customer_id
LEFT JOIN shipping_addresses s ON s.order_id = o.id;
```

## Quick Decision Guide

| Scenario | Use |
|---|---|
| You only want matched rows | INNER JOIN |
| You want all left-table rows, matched or not | LEFT JOIN |
| Finding rows with no match in another table | LEFT JOIN + WHERE right.id IS NULL |
| Optional relationship (nullable FK) | LEFT JOIN |
| Required relationship (non-null FK) | INNER JOIN |

## Summary

Use `INNER JOIN` when you only want rows with matches in both tables. Use `LEFT JOIN` when you need all rows from the left table regardless of whether a match exists. Mixing them up is one of the most common sources of subtle data errors in SQL queries - always verify your join type against the business requirement.
