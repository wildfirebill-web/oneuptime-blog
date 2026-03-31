# How to Use NOT EXISTS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, NOT EXISTS, Subqueries, SQL, Anti-Join

Description: Learn how to use NOT EXISTS in MySQL to find rows that have no matching rows in a related table, commonly used for anti-join patterns and data quality checks.

---

## What Is NOT EXISTS?

`NOT EXISTS` returns `TRUE` when the subquery returns no rows, and `FALSE` when it returns at least one row. It is the inverse of `EXISTS` and is used to find rows that have no corresponding rows in a related table - a pattern known as an "anti-join."

```sql
-- Find customers who have never placed an order
SELECT c.id, c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

## Basic NOT EXISTS Pattern

```sql
-- Template for anti-join with NOT EXISTS
SELECT *
FROM parent_table p
WHERE NOT EXISTS (
    SELECT 1
    FROM child_table c
    WHERE c.foreign_key = p.id
    -- Optional: add more conditions here
);
```

## Practical Use Cases

### Find Orphaned / Unused Records

```sql
-- Products never ordered
SELECT id, product_name, price
FROM products p
WHERE NOT EXISTS (
    SELECT 1 FROM order_items oi WHERE oi.product_id = p.id
);

-- Categories with no active products
SELECT c.id, c.name
FROM categories c
WHERE NOT EXISTS (
    SELECT 1 FROM products p
    WHERE p.category_id = c.id AND p.is_active = 1
);
```

### Finding Missing Relationships

```sql
-- Employees with no performance review this year
SELECT e.id, e.name, e.department
FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM performance_reviews r
    WHERE r.employee_id = e.id
      AND YEAR(r.review_date) = YEAR(CURDATE())
);

-- Customers who have not purchased in the last 90 days
SELECT c.id, c.name, c.email
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.id
      AND o.order_date >= DATE_SUB(CURDATE(), INTERVAL 90 DAY)
);
```

### Data Quality Checks

```sql
-- Find orders with no associated order items (data integrity issue)
SELECT o.id, o.order_date, o.customer_id
FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM order_items oi WHERE oi.order_id = o.id
);

-- Find user accounts with no login in the past year
SELECT u.id, u.username, u.created_at
FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM login_events le
    WHERE le.user_id = u.id
      AND le.logged_in_at >= DATE_SUB(NOW(), INTERVAL 1 YEAR)
);
```

## NOT EXISTS in DELETE Operations

```sql
-- Clean up: remove categories that have no products
DELETE FROM categories
WHERE NOT EXISTS (
    SELECT 1 FROM products WHERE products.category_id = categories.id
);

-- Remove duplicate rows keeping only the lowest ID
DELETE FROM contacts c1
WHERE NOT EXISTS (
    SELECT 1 FROM (
        SELECT MIN(id) AS min_id FROM contacts GROUP BY email
    ) keep_ids
    WHERE keep_ids.min_id = c1.id
);
```

## NOT EXISTS vs NOT IN - Important NULL Difference

`NOT IN` has a subtle NULL trap that `NOT EXISTS` avoids:

```sql
-- Safe: NOT EXISTS handles NULLs correctly
SELECT name FROM departments d
WHERE NOT EXISTS (
    SELECT 1 FROM employees e WHERE e.department_id = d.id
);

-- DANGEROUS: If employees.department_id contains any NULL,
-- NOT IN returns 0 rows (because NULL comparisons are unknown)
SELECT name FROM departments
WHERE id NOT IN (SELECT department_id FROM employees);
-- If ANY department_id in employees is NULL, this returns empty!

-- Fix for NOT IN: add IS NOT NULL
SELECT name FROM departments
WHERE id NOT IN (
    SELECT department_id FROM employees WHERE department_id IS NOT NULL
);
```

## NOT EXISTS with Additional Subquery Conditions

```sql
-- Customers who have never cancelled an order AND have placed at least one order
SELECT c.id, c.name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
)
AND NOT EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.id AND o.status = 'cancelled'
);
```

## Summary

`NOT EXISTS` is the most reliable way to implement anti-join queries in MySQL - finding records that have no matching rows in a related table. Unlike `NOT IN`, it handles NULL values correctly and does not silently return empty results when the subquery contains NULLs. Use it for finding orphaned records, identifying missing relationships, data quality checks, and targeting inactive users for re-engagement campaigns.
