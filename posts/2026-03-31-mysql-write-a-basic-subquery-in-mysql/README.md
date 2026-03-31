# How to Write a Basic Subquery in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Subquery, Nested Query, SQL, Database

Description: Learn how to write basic subqueries in MySQL, including scalar, row, table, and correlated subqueries with practical examples.

---

## Introduction

A subquery (also called a nested query or inner query) is a SELECT statement embedded inside another SQL statement. Subqueries allow you to use the result of one query as input for another, enabling complex filtering, calculations, and data retrieval without temporary tables.

## Types of Subqueries

1. **Scalar subquery**: returns a single value.
2. **Row subquery**: returns a single row.
3. **Table subquery**: returns multiple rows (used with IN, EXISTS, etc.).
4. **Correlated subquery**: references columns from the outer query.

## Scalar Subquery

A scalar subquery returns exactly one row and one column. It can be used anywhere a single value is expected.

```sql
-- Find employees earning more than the average salary
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

```sql
-- Add the company average salary as a column
SELECT
  name,
  salary,
  (SELECT AVG(salary) FROM employees) AS company_avg
FROM employees;
```

## Subquery in WHERE with IN

Use a subquery with `IN` to filter against a list of values:

```sql
-- Find customers who placed orders in the last 30 days
SELECT id, name, email
FROM customers
WHERE id IN (
  SELECT DISTINCT customer_id
  FROM orders
  WHERE order_date >= DATE_SUB(NOW(), INTERVAL 30 DAY)
);
```

## Subquery in WHERE with Comparison

```sql
-- Find the most recent order date
SELECT id, customer_id, order_date
FROM orders
WHERE order_date = (SELECT MAX(order_date) FROM orders);
```

## Subquery in FROM (Derived Table)

A subquery in the `FROM` clause acts as a derived table (inline view):

```sql
SELECT dept_name, avg_salary
FROM (
  SELECT department AS dept_name, AVG(salary) AS avg_salary
  FROM employees
  GROUP BY department
) AS dept_averages
WHERE avg_salary > 60000;
```

The subquery result is treated like a regular table and must have an alias.

## Subquery in SELECT Clause

A scalar subquery in the SELECT clause computes a value for each row:

```sql
SELECT
  c.name AS customer,
  (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) AS order_count
FROM customers c;
```

## Correlated Subquery

A correlated subquery references a column from the outer query and is re-evaluated for each row.

```sql
-- Find employees earning more than their department average
SELECT name, department, salary
FROM employees e1
WHERE salary > (
  SELECT AVG(salary)
  FROM employees e2
  WHERE e2.department = e1.department
);
```

Here `e1.department` connects the outer and inner queries.

## Subquery with EXISTS

```sql
-- Find categories that have at least one product
SELECT id, name
FROM categories c
WHERE EXISTS (
  SELECT 1 FROM products p WHERE p.category_id = c.id
);
```

## Subquery in UPDATE

```sql
-- Set a flag for customers who have no orders
UPDATE customers c
SET has_orders = 0
WHERE c.id NOT IN (SELECT DISTINCT customer_id FROM orders);
```

## Subquery in INSERT

```sql
-- Insert a row using a value from a subquery
INSERT INTO audit_log (action, user_id, timestamp)
VALUES ('login', (SELECT id FROM users WHERE email = 'user@example.com'), NOW());
```

## Subquery vs JOIN

Subqueries are often interchangeable with JOINs:

```sql
-- With subquery
SELECT name FROM customers
WHERE id IN (SELECT customer_id FROM orders WHERE status = 'completed');

-- With JOIN
SELECT DISTINCT c.name FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE o.status = 'completed';
```

JOINs are often faster for large datasets because MySQL can use indexes more efficiently.

## Performance Tips

- Use `EXISTS` instead of `IN` for large subquery results.
- Add indexes on the columns referenced in the subquery WHERE clause.
- Use `EXPLAIN` to check if the subquery is being optimized into a join:

```sql
EXPLAIN
SELECT name FROM customers
WHERE id IN (SELECT customer_id FROM orders WHERE status = 'completed');
```

## Summary

Subqueries let you embed one SELECT inside another to filter, compute, or derive data. Scalar subqueries return a single value, table subqueries return multiple rows for use with IN or EXISTS, and correlated subqueries reference the outer query. For performance-critical queries, consider rewriting subqueries as JOINs and always check with EXPLAIN.
