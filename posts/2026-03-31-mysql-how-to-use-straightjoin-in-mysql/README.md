# How to Use STRAIGHT_JOIN in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, STRAIGHT_JOIN, Query Optimization, JOIN, Optimizer Hints

Description: Learn how STRAIGHT_JOIN forces MySQL to join tables in a specific order, when to use it, and how to verify it improves query performance.

---

## What Is STRAIGHT_JOIN

`STRAIGHT_JOIN` is a MySQL optimizer hint that forces the query to join tables in the exact order they appear in the FROM clause. Normally, MySQL's query optimizer reorders joins to minimize the estimated cost. STRAIGHT_JOIN overrides this behavior.

It exists as both a query modifier (between SELECT and the column list) and as a join type (replacing JOIN).

## When to Use STRAIGHT_JOIN

Use STRAIGHT_JOIN when:
- The optimizer chooses a suboptimal join order that you know is wrong
- The optimizer's table statistics are stale or inaccurate
- You need deterministic join order for consistent query performance
- EXPLAIN shows a poor join order despite having good indexes

**Do not use STRAIGHT_JOIN routinely.** It disables optimizer flexibility. Always benchmark first.

## Syntax Forms

### Form 1: SELECT STRAIGHT_JOIN (All Joins)

Forces all joins in the query to follow declaration order:

```sql
SELECT STRAIGHT_JOIN
  o.id,
  c.name,
  p.title
FROM orders o
JOIN customers c ON c.id = o.customer_id
JOIN products p ON p.id = o.product_id
WHERE o.status = 'pending';
```

### Form 2: JOIN STRAIGHT_JOIN (Single Join)

Forces only a specific join to follow order; other joins are still optimized:

```sql
SELECT
  o.id,
  c.name
FROM customers c
STRAIGHT_JOIN orders o ON o.customer_id = c.id
WHERE c.region = 'US';
```

## A Practical Example

Consider a query where the optimizer chooses the wrong driving table:

```sql
-- Table sizes:
-- orders: 50,000,000 rows
-- customers: 10,000 rows
-- order_items: 200,000,000 rows

-- Optimizer mistakenly starts with order_items (large table)
EXPLAIN SELECT
  c.name,
  COUNT(*) AS item_count
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
WHERE c.region = 'EU';
```

```text
+----+...+-------------+------+------+...+
| id |...| table       | type | rows |...|
+----+...+-------------+------+------+...+
|  1 |...| order_items | ALL  | 200M |...|
|  1 |...| orders      | ref  | 1    |...|
|  1 |...| customers   | ref  | 1    |...|
```

```sql
-- Force optimizer to start with customers (small, filtered by region)
SELECT STRAIGHT_JOIN
  c.name,
  COUNT(*) AS item_count
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
WHERE c.region = 'EU';
```

```text
+----+...+-------------+------+------+...+
| id |...| table       | type | rows |...|
+----+...+-------------+------+------+...+
|  1 |...| customers   | ref  | 500  |...|
|  1 |...| orders      | ref  | 5    |...|
|  1 |...| order_items | ref  | 10   |...|
```

The second plan starts with `customers` filtered to the EU region and works outward, dramatically reducing the rows examined.

## Comparing Plans with EXPLAIN

Always verify the impact:

```sql
-- Without STRAIGHT_JOIN
EXPLAIN FORMAT=JSON
SELECT c.name, COUNT(*) AS item_count
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
WHERE c.region = 'EU'\G

-- With STRAIGHT_JOIN
EXPLAIN FORMAT=JSON
SELECT STRAIGHT_JOIN c.name, COUNT(*) AS item_count
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
WHERE c.region = 'EU'\G
```

Compare `rows_examined_per_scan` in the JSON output.

## Modern Alternative: Optimizer Hints

MySQL 8.0+ provides more granular optimizer hints that are preferred over STRAIGHT_JOIN:

```sql
-- JOIN_ORDER hint: specify join order without fixing all optimizer decisions
SELECT /*+ JOIN_ORDER(c, o, oi) */
  c.name,
  COUNT(*) AS item_count
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
WHERE c.region = 'EU';

-- JOIN_PREFIX: fix only the first table(s) in the join order
SELECT /*+ JOIN_PREFIX(c) */
  c.name,
  COUNT(*) AS item_count
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
WHERE c.region = 'EU';
```

Optimizer hints are more expressive and do not fully disable the optimizer.

## Updating Statistics Before Using STRAIGHT_JOIN

Before resorting to STRAIGHT_JOIN, ensure statistics are current:

```sql
ANALYZE TABLE customers;
ANALYZE TABLE orders;
ANALYZE TABLE order_items;
```

Stale statistics often cause the optimizer to choose wrong join orders. Refreshing statistics may eliminate the need for STRAIGHT_JOIN entirely.

## Summary

`STRAIGHT_JOIN` forces MySQL to join tables in the declared order, bypassing the optimizer's join reordering. It is useful when the optimizer selects a suboptimal driving table due to stale statistics or complex join conditions. Always verify with EXPLAIN before and after, consider refreshing statistics first, and prefer MySQL 8.0's `JOIN_ORDER` hint for more controlled optimization.
