# How to Use Scalar Subqueries in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Subquery, Database, Query

Description: Learn how scalar subqueries work in MySQL, how to write them in SELECT, WHERE, and HAVING clauses, and when to use them versus joins for single-value lookups.

---

A scalar subquery is a subquery that returns exactly one row and one column: a single value. MySQL allows scalar subqueries anywhere a single value expression is valid, including the `SELECT` list, `WHERE` clause, `HAVING` clause, and `ORDER BY` clause.

## What makes a subquery scalar

```sql
-- Returns one row, one column: scalar
SELECT MAX(salary) FROM employees;  -- e.g. 95000

-- Returns multiple rows: NOT scalar (will cause error if used as scalar)
SELECT salary FROM employees;
```

If a scalar subquery returns more than one row at runtime, MySQL raises:
`ERROR 1242 (21000): Subquery returns more than 1 row`

## Scalar subquery in SELECT list

The most common use: look up a single related value for each row in the outer query.

```sql
CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    name          VARCHAR(60)
);

CREATE TABLE employees (
    employee_id   INT PRIMARY KEY,
    name          VARCHAR(100),
    salary        DECIMAL(10,2),
    department_id INT
);

-- Show each employee with their department's average salary
SELECT
    e.name,
    e.salary,
    (
        SELECT ROUND(AVG(e2.salary), 2)
        FROM employees e2
        WHERE e2.department_id = e.department_id
    ) AS dept_avg_salary
FROM employees e;
```

This is a correlated scalar subquery: the inner query references `e.department_id` from the outer row.

## Scalar subquery in WHERE clause

```sql
-- Find employees earning more than the overall average
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

```sql
-- Find the employee with the highest salary
SELECT name, salary
FROM employees
WHERE salary = (SELECT MAX(salary) FROM employees);
```

## Scalar subquery in HAVING clause

```sql
-- Find departments whose average salary exceeds the company-wide average
SELECT department_id, AVG(salary) AS dept_avg
FROM employees
GROUP BY department_id
HAVING AVG(salary) > (SELECT AVG(salary) FROM employees);
```

## Scalar subquery in ORDER BY clause

```sql
-- Order departments by their employee count (scalar lookup)
SELECT d.name
FROM departments d
ORDER BY (
    SELECT COUNT(*) FROM employees e WHERE e.department_id = d.department_id
) DESC;
```

## Returning a scalar from a joined subquery

Any subquery that ultimately returns one row and one column qualifies:

```sql
SELECT
    e.name,
    (
        SELECT d.name
        FROM departments d
        WHERE d.department_id = e.department_id
        LIMIT 1
    ) AS department_name
FROM employees e;
```

`LIMIT 1` guards against cases where the inner query could return more than one row.

## NULL behaviour

If the scalar subquery returns no rows, the result is `NULL`:

```sql
-- Returns NULL if the employee has no department
SELECT
    e.name,
    (SELECT d.name FROM departments d WHERE d.department_id = e.department_id) AS dept
FROM employees e;
```

## Scalar subquery vs JOIN

Both approaches can look up a single related value:

```sql
-- Scalar subquery (correlated, runs once per outer row)
SELECT e.name, (SELECT d.name FROM departments d WHERE d.department_id = e.department_id) AS dept
FROM employees e;

-- JOIN (usually more efficient for large result sets)
SELECT e.name, d.name AS dept
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;
```

For large tables, the `JOIN` form is usually faster because MySQL can use hash joins and execute the lookup once per unique key rather than once per row. Use scalar subqueries when:
- You need an aggregate value that changes per outer row.
- A join would produce duplicate rows that are hard to de-duplicate.
- Readability matters more than optimal performance on a small table.

## Performance check with EXPLAIN

```sql
EXPLAIN
SELECT
    e.name,
    e.salary,
    (SELECT AVG(e2.salary) FROM employees e2 WHERE e2.department_id = e.department_id) AS dept_avg
FROM employees e\G
```

A correlated scalar subquery that runs once per outer row can be slow on large tables. Consider rewriting as a JOIN with a derived table if performance is a concern.

## Summary

Scalar subqueries return a single value and can be placed in `SELECT`, `WHERE`, `HAVING`, or `ORDER BY`. They are convenient for per-row lookups and aggregate comparisons. When a scalar subquery is correlated and the outer table is large, rewrite it as a `JOIN` with a pre-aggregated derived table for better performance.
