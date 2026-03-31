# How to Optimize EXISTS vs IN Performance in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Query, Optimization, Subquery, Performance

Description: Learn when to use EXISTS vs IN in MySQL, how each performs differently depending on table sizes and indexes, and what the optimizer does with each pattern.

---

## EXISTS and IN Both Test Membership

Both `EXISTS` and `IN` with a subquery test whether rows in the outer query have matching rows in the subquery. Their performance differs based on subquery size, result set size, and index availability.

```sql
-- IN with subquery
SELECT * FROM customers
WHERE id IN (SELECT customer_id FROM orders WHERE total > 1000);

-- EXISTS equivalent
SELECT * FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders o
  WHERE o.customer_id = c.id AND o.total > 1000
);
```

## How MySQL Optimizes IN

In MySQL 8.0, the optimizer automatically converts many `IN (subquery)` patterns to semijoin joins. You can verify this:

```sql
EXPLAIN SELECT * FROM customers
WHERE id IN (SELECT customer_id FROM orders WHERE total > 1000);
SHOW WARNINGS;
-- May show: semijoin transformation in the rewritten query
```

If the semijoin transformation is applied, performance is similar to an explicit JOIN.

## When EXISTS Performs Better

`EXISTS` terminates as soon as one matching row is found. It is better when:
- The outer table is small and the inner table is large
- You expect most outer rows to have at least one match (stops early on first match)

```sql
-- Better with EXISTS: outer table is small (10 VIP customers)
-- Inner table is large (5 million orders)
SELECT * FROM vip_customers c
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

## When IN Performs Better

`IN` is better when:
- The subquery result set is small and can be materialized once
- The subquery is not correlated (does not reference the outer table)

```sql
-- Better with IN: small subquery result materialized once
SELECT * FROM products
WHERE category_id IN (SELECT id FROM categories WHERE active = 1);
-- categories table is small; result is materialized and reused
```

## NOT EXISTS vs NOT IN (Important Difference)

`NOT IN` and `NOT EXISTS` behave differently with NULL values:

```sql
-- NOT IN with NULL in subquery returns no rows (silent bug!)
SELECT * FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);
-- If any order has customer_id = NULL, this returns ZERO rows

-- NOT EXISTS handles NULL correctly
SELECT * FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

Always prefer `NOT EXISTS` over `NOT IN` when NULL values are possible in the subquery column.

## Rewrite as JOIN for Best Control

For predictable performance, rewrite both EXISTS and IN as explicit JOINs (or LEFT JOIN for NOT EXISTS):

```sql
-- IN rewritten as JOIN
SELECT DISTINCT c.*
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE o.total > 1000;

-- NOT EXISTS rewritten as LEFT JOIN
SELECT c.*
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.customer_id IS NULL;
```

Explicit JOINs give you full control over join order and index use.

## Verify with EXPLAIN

```sql
-- Compare execution plans
EXPLAIN SELECT * FROM products
WHERE category_id IN (SELECT id FROM categories WHERE active = 1);

EXPLAIN SELECT * FROM products p
WHERE EXISTS (
  SELECT 1 FROM categories c WHERE c.id = p.category_id AND c.active = 1
);
```

Look for `DEPENDENT SUBQUERY` in the `select_type` column - this indicates the subquery re-executes per outer row and may need optimization.

## Summary

In modern MySQL (8.0+), the optimizer handles many EXISTS and IN patterns equivalently through semijoin transformation. The practical differences emerge with NULL values (always use NOT EXISTS instead of NOT IN), and with highly asymmetric table sizes where EXISTS's early-exit behavior matters. When performance is critical, rewrite both patterns as explicit JOINs for the most predictable and controllable execution plan.
