# What Is a Correlated Subquery in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Subqueries, Query Optimization, Advanced Sql

Description: A correlated subquery in MySQL is a subquery that references columns from the outer query and is re-evaluated for each row processed by the outer query.

---

## Overview

A correlated subquery is a subquery that references one or more columns from the outer query. Unlike a regular (non-correlated) subquery that executes once and returns a result set, a correlated subquery is re-executed once for every row in the outer query. This makes it powerful but potentially slow on large datasets.

## Basic Example

```sql
-- Find employees who earn more than the average salary in their department
SELECT name, salary, department_id
FROM employees e1
WHERE salary > (
  SELECT AVG(salary)
  FROM employees e2
  WHERE e2.department_id = e1.department_id  -- correlation: references outer table
);
```

The subquery is correlated because it references `e1.department_id` from the outer query.

## Correlated vs Non-Correlated Subquery

```sql
-- Non-correlated: executes once
SELECT name FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- Correlated: executes once per outer row
SELECT name FROM employees e
WHERE salary > (
  SELECT AVG(salary) FROM employees
  WHERE department_id = e.department_id
);
```

## Common Use Cases

### EXISTS Pattern

```sql
-- Find customers who have placed at least one order
SELECT c.name
FROM customers c
WHERE EXISTS (
  SELECT 1
  FROM orders o
  WHERE o.customer_id = c.id
);
```

### NOT EXISTS Pattern

```sql
-- Find customers who have never placed an order
SELECT c.name
FROM customers c
WHERE NOT EXISTS (
  SELECT 1
  FROM orders o
  WHERE o.customer_id = c.id
);
```

### Finding Rows Without a Match

```sql
-- Products never ordered (using NOT EXISTS)
SELECT p.id, p.name
FROM products p
WHERE NOT EXISTS (
  SELECT 1 FROM order_items oi
  WHERE oi.product_id = p.id
);
```

### Scalar Correlated Subquery in SELECT

```sql
-- Show each employee with their department's average salary
SELECT
  name,
  salary,
  (SELECT AVG(salary)
   FROM employees e2
   WHERE e2.department_id = e1.department_id) AS dept_avg
FROM employees e1;
```

## Performance Considerations

Correlated subqueries can be slow because they run once per outer row. Check with EXPLAIN:

```sql
EXPLAIN SELECT * FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

Look for `DEPENDENT SUBQUERY` in the `select_type` column - this confirms correlation.

## Rewriting as a JOIN (Often Faster)

```sql
-- Correlated subquery (potentially slow)
SELECT c.name
FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id
);

-- Equivalent JOIN (usually faster)
SELECT DISTINCT c.name
FROM customers c
JOIN orders o ON o.customer_id = c.id;
```

## Rewriting as a Derived Table

```sql
-- Correlated subquery
SELECT e.name, e.salary,
  (SELECT AVG(salary) FROM employees WHERE department_id = e.department_id) AS dept_avg
FROM employees e;

-- Equivalent with derived table (executes once)
SELECT e.name, e.salary, d.dept_avg
FROM employees e
JOIN (
  SELECT department_id, AVG(salary) AS dept_avg
  FROM employees
  GROUP BY department_id
) AS d ON d.department_id = e.department_id;
```

## When Correlated Subqueries Are Acceptable

- Small outer result sets (few rows)
- EXISTS/NOT EXISTS checks where the subquery returns quickly via index
- Queries where the logic cannot be easily expressed as a JOIN

## Summary

Correlated subqueries reference columns from the outer query and are re-evaluated per outer row. They are useful for patterns like EXISTS, NOT EXISTS, and per-row aggregations, but can be slow on large datasets. Always check the execution plan and consider rewriting as JOINs or derived tables when performance is a concern.
