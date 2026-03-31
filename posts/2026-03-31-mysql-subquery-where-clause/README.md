# How to Use Subqueries in the WHERE Clause in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Subquery, Database, Query

Description: Learn how to use subqueries in the MySQL WHERE clause with IN, NOT IN, EXISTS, comparison operators, and correlated subqueries to filter rows dynamically.

---

Subqueries in the `WHERE` clause let you filter rows based on values computed from another query. MySQL supports scalar, list, and correlated subqueries in `WHERE`.

## Scalar comparison: a single value

```sql
CREATE TABLE employees (
    employee_id   INT PRIMARY KEY,
    name          VARCHAR(100),
    salary        DECIMAL(10,2),
    department_id INT
);

-- Find employees earning more than the company average
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

The subquery `(SELECT AVG(salary) FROM employees)` runs once and returns a single number. The outer query compares each row's salary against that number.

## IN: match against a list of values

```sql
CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    name          VARCHAR(60),
    location      VARCHAR(60)
);

-- Find employees in departments located in London
SELECT name
FROM employees
WHERE department_id IN (
    SELECT department_id
    FROM departments
    WHERE location = 'London'
);
```

`IN` is equivalent to `= ANY(subquery)`. The subquery can return multiple rows.

## NOT IN: exclude a list

```sql
-- Find employees NOT in any London department
SELECT name
FROM employees
WHERE department_id NOT IN (
    SELECT department_id
    FROM departments
    WHERE location = 'London'
);
```

Caution: if the subquery returns any `NULL` values, `NOT IN` returns no rows for the outer query. Use `NOT EXISTS` or filter NULLs from the subquery to avoid this trap.

```sql
-- Safe form
SELECT name
FROM employees
WHERE department_id NOT IN (
    SELECT department_id
    FROM departments
    WHERE location = 'London'
      AND department_id IS NOT NULL
);
```

## EXISTS: check for the existence of related rows

```sql
-- Find employees who have submitted at least one expense report
SELECT e.name
FROM employees e
WHERE EXISTS (
    SELECT 1
    FROM expense_reports er
    WHERE er.employee_id = e.employee_id
);
```

`EXISTS` returns `TRUE` as soon as the inner query finds one matching row. It does not care about column values, so `SELECT 1` is conventional and efficient.

## NOT EXISTS: find rows with no related records

```sql
-- Find employees with no expense reports
SELECT e.name
FROM employees e
WHERE NOT EXISTS (
    SELECT 1
    FROM expense_reports er
    WHERE er.employee_id = e.employee_id
);
```

This is the safest anti-join form because it handles `NULL` values correctly.

## Comparison operators: =, <>, <, >, <=, >=

```sql
-- Find the employee with the maximum salary
SELECT name, salary
FROM employees
WHERE salary = (SELECT MAX(salary) FROM employees);

-- Find employees earning less than the minimum in department 5
SELECT name, salary
FROM employees
WHERE salary < (
    SELECT MIN(salary)
    FROM employees
    WHERE department_id = 5
);
```

## ANY and ALL with WHERE

```sql
-- Find employees earning more than at least one employee in department 3
SELECT name, salary
FROM employees
WHERE salary > ANY (
    SELECT salary FROM employees WHERE department_id = 3
);

-- Find employees earning more than every employee in department 3
SELECT name, salary
FROM employees
WHERE salary > ALL (
    SELECT salary FROM employees WHERE department_id = 3
);
```

## Correlated subquery: referencing the outer row

A correlated subquery references columns from the outer query. It runs once for each outer row.

```sql
-- Find employees earning more than the average in their own department
SELECT name, salary, department_id
FROM employees e
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
    WHERE department_id = e.department_id
);
```

The inner query changes its result for each value of `e.department_id`.

## Multiple conditions in WHERE with subqueries

```sql
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees)
  AND department_id IN (
      SELECT department_id FROM departments WHERE location = 'London'
  );
```

## Performance considerations

- Scalar subqueries in `WHERE` execute once (non-correlated) or once per row (correlated).
- Index the join column used in a correlated subquery for best performance.
- For large tables, `IN (subquery)` may be rewritten by MySQL as a semi-join, which can be very efficient.
- Use `EXPLAIN` to check the actual execution plan.

```sql
EXPLAIN
SELECT name FROM employees
WHERE department_id IN (SELECT department_id FROM departments WHERE location = 'London')\G
```

## Summary

Subqueries in the `WHERE` clause support scalar comparisons, `IN`, `NOT IN`, `EXISTS`, `NOT EXISTS`, and `ANY`/`ALL`. Prefer `NOT EXISTS` over `NOT IN` to avoid NULL-related surprises. Index the columns used in correlated subqueries, and use `EXPLAIN` to verify that MySQL is applying an efficient semi-join or index lookup strategy.
