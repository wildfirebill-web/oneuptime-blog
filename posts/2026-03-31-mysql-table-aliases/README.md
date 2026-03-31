# How to Use Table Aliases in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Table Alias, JOIN

Description: Learn how to use table aliases in MySQL queries to shorten table names, resolve ambiguous column names in JOINs, and enable self-joins.

---

Table aliases assign a short name to a table within a query. They reduce repetitive typing, resolve column name ambiguity in joins, and are required for self-joins where the same table appears more than once.

## Basic Table Alias

```sql
-- Without alias: verbose
SELECT employees.employee_id, employees.name, employees.department
FROM employees;

-- With alias: shorter
SELECT e.employee_id, e.name, e.department
FROM employees AS e;
```

Like column aliases, the `AS` keyword is optional but recommended:

```sql
-- Both are equivalent
FROM employees AS e
FROM employees e
```

## Aliases in JOIN Queries

Table aliases are essential in joins to disambiguate columns that exist in multiple tables:

```sql
SELECT
  o.order_id,
  o.order_date,
  o.total,
  c.name AS customer_name,
  c.email
FROM orders AS o
INNER JOIN customers AS c
  ON o.customer_id = c.customer_id
WHERE o.status = 'shipped';
```

Without aliases, you would need to write `orders.order_id`, `customers.name`, etc. throughout the query.

## Multiple Table Aliases in Complex Joins

```sql
SELECT
  oi.quantity,
  oi.unit_price,
  p.product_name,
  p.sku,
  o.order_date,
  c.name AS customer_name
FROM order_items AS oi
INNER JOIN orders AS o ON oi.order_id = o.order_id
INNER JOIN products AS p ON oi.product_id = p.product_id
INNER JOIN customers AS c ON o.customer_id = c.customer_id
WHERE o.order_date >= '2025-01-01';
```

Consistent single-letter or abbreviated aliases make complex multi-table queries readable at a glance.

## Self-Joins with Aliases

Self-joins require aliases because you are joining a table to itself and need to distinguish the two instances:

```sql
-- Find employees and their managers (both in the same table)
SELECT
  e.name AS employee_name,
  e.department,
  m.name AS manager_name
FROM employees AS e
LEFT JOIN employees AS m
  ON e.manager_id = m.employee_id;
```

Without two different aliases (`e` and `m`), MySQL cannot tell which instance of `employees` a column reference belongs to.

## Aliases in Subqueries

Subqueries in the `FROM` clause must be given an alias:

```sql
-- The subquery result must be aliased
SELECT dept_stats.department, dept_stats.avg_salary
FROM (
  SELECT department, AVG(salary) AS avg_salary
  FROM employees
  GROUP BY department
) AS dept_stats
WHERE dept_stats.avg_salary > 60000;
```

## Aliases are Query-Scoped

An alias defined in the `FROM` clause is available throughout the query (SELECT, WHERE, ORDER BY, HAVING), but not outside it:

```sql
SELECT e.name, e.salary
FROM employees AS e
WHERE e.salary > (
  SELECT AVG(salary) FROM employees  -- no alias needed here
)
ORDER BY e.salary DESC;
```

## Naming Conventions

```text
Short and meaningful aliases improve readability:

  employees    -> e or emp
  customers    -> c or cust
  orders       -> o or ord
  order_items  -> oi
  products     -> p or prod
```

Avoid single letters for large queries with many tables - use short abbreviations instead to keep context clear.

## Summary

Table aliases give a table a short name for the duration of a query. Use `FROM table_name AS alias` (the `AS` is optional). Aliases are essential in JOIN queries to avoid ambiguous column references, and required for self-joins and subquery results in the `FROM` clause. Consistent short abbreviations improve readability in complex multi-table queries.
