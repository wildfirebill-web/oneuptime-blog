# How to Write a Basic Subquery in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Subquery, Query, Nested Query, SELECT

Description: Learn how to write basic subqueries in MySQL, including scalar subqueries, row subqueries, and derived tables in the FROM clause.

---

## What Is a Subquery

A subquery is a `SELECT` statement nested inside another SQL statement. The outer query uses the result of the inner query (the subquery) as a value, a list of values, or a temporary table.

Subqueries can appear in the `SELECT` list, `FROM` clause, `WHERE` clause, and `HAVING` clause.

## Scalar Subquery - Single Value

A scalar subquery returns exactly one row and one column. It can be used anywhere a single value is expected:

```sql
-- Find products priced above the average price
SELECT id, name, price
FROM products
WHERE price > (SELECT AVG(price) FROM products);
```

The subquery `SELECT AVG(price) FROM products` is evaluated once and returns a single number.

## Subquery in the SELECT List

Return the department name alongside employee records without a JOIN:

```sql
SELECT
  e.id,
  e.name,
  (SELECT d.name FROM departments d WHERE d.id = e.department_id) AS department_name
FROM employees e;
```

Note: This type of correlated subquery runs once per row. For large tables, a `JOIN` is more efficient.

## Subquery in the WHERE Clause

Find all employees who work in departments located in New York:

```sql
SELECT id, name
FROM employees
WHERE department_id IN (
  SELECT id FROM departments WHERE location = 'New York'
);
```

## Subquery in the FROM Clause - Derived Table

A subquery in the `FROM` clause acts as a temporary table (also called a derived table):

```sql
SELECT dept, avg_salary
FROM (
  SELECT department AS dept, AVG(salary) AS avg_salary
  FROM employees
  GROUP BY department
) AS dept_summary
WHERE avg_salary > 60000;
```

The derived table must be given an alias (`dept_summary` in this example).

## Correlated Subquery

A correlated subquery references columns from the outer query. It is re-evaluated for each row of the outer query:

```sql
-- Find employees who earn more than the average for their department
SELECT id, name, salary, department_id
FROM employees e
WHERE salary > (
  SELECT AVG(salary)
  FROM employees
  WHERE department_id = e.department_id
);
```

Correlated subqueries can be slow on large tables. Consider rewriting them as a JOIN against a derived table.

## Rewriting a Correlated Subquery as a JOIN

```sql
SELECT e.id, e.name, e.salary
FROM employees e
JOIN (
  SELECT department_id, AVG(salary) AS avg_salary
  FROM employees
  GROUP BY department_id
) dept_avg ON e.department_id = dept_avg.department_id
WHERE e.salary > dept_avg.avg_salary;
```

## Checking Subquery Performance

Use `EXPLAIN` to see how MySQL executes a subquery:

```sql
EXPLAIN
SELECT id, name
FROM employees
WHERE department_id IN (
  SELECT id FROM departments WHERE location = 'New York'
);
```

## Summary

Subqueries let you nest one SELECT inside another to filter, calculate, or build temporary result sets. Scalar subqueries return a single value, list subqueries work with `IN`, and derived tables act as virtual tables in the `FROM` clause. For correlated subqueries on large datasets, consider rewriting as a `JOIN` for better performance.
