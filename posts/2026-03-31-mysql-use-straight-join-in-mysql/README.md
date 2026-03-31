# How to Use Straight Join in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Straight Join, Query Optimization, Sql, Performance

Description: Learn how to use STRAIGHT_JOIN in MySQL to override the optimizer's join order and force a specific table to be read first when the optimizer chooses a suboptimal plan.

---

## Introduction

`STRAIGHT_JOIN` is a MySQL-specific join hint that forces the optimizer to join tables in the exact order they appear in the query, from left to right. Normally, MySQL's optimizer decides the most efficient join order based on statistics. `STRAIGHT_JOIN` overrides this when you know a better order based on your data distribution.

## Basic Syntax

As a hint after SELECT:

```sql
SELECT STRAIGHT_JOIN a.col1, b.col2
FROM table_a a
JOIN table_b b ON a.id = b.a_id;
```

Or as a join type:

```sql
SELECT a.col1, b.col2
FROM table_a a
STRAIGHT_JOIN table_b b ON a.id = b.a_id;
```

## Normal JOIN vs STRAIGHT_JOIN

With a normal JOIN, MySQL's optimizer chooses the join order:

```sql
-- Optimizer may read table_b first if it estimates that's more efficient
SELECT a.name, b.order_count
FROM customers a
JOIN order_stats b ON a.id = b.customer_id;
```

With `STRAIGHT_JOIN`, MySQL reads `customers` first, then looks up in `order_stats`:

```sql
-- Forces: read customers first, then join order_stats
SELECT a.name, b.order_count
FROM customers a
STRAIGHT_JOIN order_stats b ON a.id = b.customer_id;
```

## When to Use STRAIGHT_JOIN

Use `STRAIGHT_JOIN` when:
- `EXPLAIN` shows an inefficient join order.
- The optimizer picks a large table as the driving table when a smaller filtered set would be better.
- You have domain knowledge that a specific order will be faster (e.g., one table has a highly selective WHERE clause).

## Diagnosing Join Order with EXPLAIN

```sql
EXPLAIN
SELECT c.name, o.total
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending';
```

If the output shows a large `rows` estimate for the first table and the optimizer is picking the wrong driving table, consider `STRAIGHT_JOIN`.

## Example: Forcing a Better Join Order

Suppose `orders` has a very selective filter and `customers` is large:

```sql
-- Optimizer might read customers first (bad choice: full scan of large table)
SELECT c.name, o.total
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE o.status = 'pending' AND o.amount > 10000;

-- STRAIGHT_JOIN forces orders to be read first (good: small filtered set)
SELECT STRAIGHT_JOIN c.name, o.total
FROM orders o
STRAIGHT_JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'pending' AND o.amount > 10000;
```

## STRAIGHT_JOIN with Multiple Tables

The hint applies to all joins in the query, forcing left-to-right order:

```sql
SELECT STRAIGHT_JOIN o.id, c.name, p.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.status = 'pending';
```

Or apply only to one specific join using the join-type syntax:

```sql
SELECT o.id, c.name, p.name
FROM orders o
STRAIGHT_JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id;
```

## Comparing with EXPLAIN

Always verify with EXPLAIN:

```sql
EXPLAIN
SELECT STRAIGHT_JOIN c.name, o.total
FROM orders o
STRAIGHT_JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'pending';
```

Check that the `rows` estimates and access types are better than the original query.

## Caveats and Best Practices

- Use `STRAIGHT_JOIN` sparingly. The optimizer is usually right.
- Revisit `STRAIGHT_JOIN` hints after adding or updating indexes, as statistics change.
- Document why the hint is used in a comment for future maintainers:

```sql
-- STRAIGHT_JOIN: orders has selective filter; read it first before customers
SELECT STRAIGHT_JOIN c.name, o.total
FROM orders o
STRAIGHT_JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'pending' AND o.amount > 10000;
```

## Summary

`STRAIGHT_JOIN` is a MySQL-specific optimization hint that forces the optimizer to join tables in the order specified in the query. Use it when `EXPLAIN` reveals the optimizer is choosing an inefficient join order and a specific driving table would reduce the number of rows processed. Always test with `EXPLAIN` before and after applying the hint.
