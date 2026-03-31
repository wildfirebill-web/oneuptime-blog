# How to Use NOT EXISTS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Not Exists, Subquery, Sql, Database

Description: Learn how to use NOT EXISTS in MySQL to filter rows where no matching record exists in a subquery, with examples for anti-joins and NULL-safe exclusion.

---

## Introduction

`NOT EXISTS` is the negation of `EXISTS` in MySQL. It returns TRUE when the subquery returns no rows, making it ideal for finding records that do NOT have matching rows in another table. It is a reliable, NULL-safe way to perform anti-join operations.

## Basic NOT EXISTS Syntax

```sql
SELECT column1
FROM table1 t1
WHERE NOT EXISTS (
  SELECT 1 FROM table2 t2 WHERE t2.foreign_key = t1.id
);
```

## Find Customers Without Orders

```sql
SELECT id, name, email
FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

This returns only customers who have never placed an order.

## Find Products Never Ordered

```sql
SELECT p.id, p.name, p.price
FROM products p
WHERE NOT EXISTS (
  SELECT 1
  FROM order_items oi
  WHERE oi.product_id = p.id
);
```

## NOT EXISTS with Additional Conditions

Find customers who have never placed an order in a specific category:

```sql
SELECT c.id, c.name
FROM customers c
WHERE NOT EXISTS (
  SELECT 1
  FROM orders o
  JOIN order_items oi ON o.id = oi.order_id
  JOIN products p ON oi.product_id = p.id
  WHERE o.customer_id = c.id
    AND p.category = 'Electronics'
);
```

## NOT EXISTS vs NOT IN

`NOT EXISTS` is generally safer than `NOT IN` when the subquery can return NULLs.

```sql
-- NOT IN: returns no rows if subquery contains any NULL (unexpected behavior)
SELECT id FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);
-- If any order has customer_id = NULL, this returns empty set!

-- NOT EXISTS: NULL-safe
SELECT id FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
-- Works correctly regardless of NULLs
```

## NOT EXISTS vs LEFT JOIN / IS NULL

Both approaches find rows without matches, but `NOT EXISTS` is often more readable:

```sql
-- Using LEFT JOIN ... IS NULL (anti-join pattern)
SELECT c.id, c.name
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.customer_id IS NULL;

-- Using NOT EXISTS
SELECT c.id, c.name
FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

Both are functionally equivalent. MySQL's optimizer may transform one into the other internally.

## NOT EXISTS in DELETE

Delete records from a staging table where the corresponding production record does not exist:

```sql
DELETE FROM staging_orders s
WHERE NOT EXISTS (
  SELECT 1 FROM customers c WHERE c.id = s.customer_id
);
```

## NOT EXISTS in UPDATE

Mark products as discontinued if they have no recent orders:

```sql
UPDATE products p
SET status = 'discontinued'
WHERE NOT EXISTS (
  SELECT 1
  FROM order_items oi
  JOIN orders o ON oi.order_id = o.id
  WHERE oi.product_id = p.id
    AND o.order_date >= DATE_SUB(NOW(), INTERVAL 1 YEAR)
);
```

## NOT EXISTS to Find Missing Assignments

Find employees without a manager assignment:

```sql
SELECT e.id, e.name
FROM employees e
WHERE NOT EXISTS (
  SELECT 1
  FROM manager_assignments ma
  WHERE ma.employee_id = e.id
    AND ma.end_date IS NULL
);
```

## Performance Considerations

Index the column referenced in the subquery WHERE clause for fast lookups:

```sql
CREATE INDEX idx_orders_customer ON orders(customer_id);

EXPLAIN
SELECT id, name FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

MySQL should show an index lookup on the `orders` table in the execution plan.

## NOT EXISTS with Correlated Subquery

The subquery in NOT EXISTS is correlated - it runs once per row of the outer query. The outer query column reference is what connects the two tables.

```sql
SELECT e.name, e.department
FROM employees e
WHERE NOT EXISTS (
  SELECT 1
  FROM performance_reviews pr
  WHERE pr.employee_id = e.id
    AND pr.review_year = YEAR(NOW())
);
-- Returns employees who haven't had a review this year
```

## Summary

`NOT EXISTS` is the preferred, NULL-safe way to find rows that have no matching record in another table. It is more reliable than `NOT IN` when NULL values may be present in the subquery result. For best performance, index the columns referenced in the correlated subquery's WHERE clause.
