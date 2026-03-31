# What Is a Scalar Subquery in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Subqueries, Advanced Sql, Query Optimization

Description: A scalar subquery in MySQL is a subquery that returns exactly one row and one column, allowing it to be used anywhere a single value expression is expected.

---

## Overview

A scalar subquery is a subquery that returns a single value - exactly one row with one column. Because it produces a single value, it can be used anywhere a literal or column expression can appear: in the `SELECT` list, `WHERE` clause, `HAVING` clause, `ORDER BY` clause, or as a function argument.

If the subquery returns more than one row, MySQL raises an error. If it returns no rows, it returns `NULL`.

## Basic Example

```sql
-- Get the name of the most expensive product
SELECT name, price
FROM products
WHERE price = (SELECT MAX(price) FROM products);
```

## Scalar Subquery in SELECT List

```sql
-- Display each order with the customer's total order count
SELECT
  o.id,
  o.amount,
  (SELECT COUNT(*) FROM orders WHERE customer_id = o.customer_id) AS total_orders
FROM orders o;
```

## Scalar Subquery in WHERE Clause

```sql
-- Find employees earning more than the company average
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

## Scalar Subquery in HAVING Clause

```sql
-- Departments with more employees than the company average per department
SELECT department_id, COUNT(*) AS emp_count
FROM employees
GROUP BY department_id
HAVING COUNT(*) > (
  SELECT AVG(dept_count)
  FROM (
    SELECT COUNT(*) AS dept_count
    FROM employees
    GROUP BY department_id
  ) AS dept_averages
);
```

## Scalar Subquery in ORDER BY

```sql
-- Sort customers by their latest order date
SELECT id, name
FROM customers c
ORDER BY (
  SELECT MAX(created_at)
  FROM orders
  WHERE customer_id = c.id
) DESC;
```

## Handling NULL Results

When a scalar subquery returns no rows, the result is NULL:

```sql
-- This returns NULL if the customer has no orders
SELECT
  c.name,
  (SELECT MAX(amount) FROM orders WHERE customer_id = c.id) AS max_order
FROM customers c;

-- Use COALESCE to substitute a default
SELECT
  c.name,
  COALESCE(
    (SELECT MAX(amount) FROM orders WHERE customer_id = c.id),
    0.00
  ) AS max_order
FROM customers c;
```

## Error: More Than One Row

```sql
-- This causes an error if multiple rows match
SELECT name FROM customers
WHERE id = (SELECT customer_id FROM orders WHERE status = 'pending');
-- ERROR 1242 (21000): Subquery returns more than 1 row

-- Fix: use IN instead
SELECT name FROM customers
WHERE id IN (SELECT customer_id FROM orders WHERE status = 'pending');
```

## Performance: Correlated vs Non-Correlated Scalar Subquery

```sql
-- Non-correlated: executes once, fast
SELECT name FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- Correlated scalar: executes once per row, can be slow
SELECT name,
  (SELECT AVG(salary) FROM employees e2 WHERE e2.dept_id = e1.dept_id) AS dept_avg
FROM employees e1;
```

For correlated scalar subqueries, rewriting as a JOIN with a derived table is usually faster:

```sql
SELECT e.name, d.dept_avg
FROM employees e
JOIN (
  SELECT dept_id, AVG(salary) AS dept_avg
  FROM employees
  GROUP BY dept_id
) AS d ON d.dept_id = e.dept_id;
```

## Scalar Subquery as a Default Value

```sql
-- Insert with a computed default
INSERT INTO audit_log (event, user_count)
VALUES ('daily_summary', (SELECT COUNT(*) FROM users WHERE active = 1));
```

## Summary

Scalar subqueries in MySQL return a single value and can appear wherever a value expression is valid - in SELECT lists, WHERE conditions, ORDER BY clauses, and more. They are concise and readable but can be slow when correlated. For performance-critical code, rewrite correlated scalar subqueries as JOIN operations with grouped derived tables.
