# How to Rewrite Subqueries as JOINs in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Subquery, Join, Performance, Database, Query

Description: Learn how to rewrite common MySQL subquery patterns as JOINs to improve readability, enable better index usage, and let the optimizer choose a more efficient execution plan.

---

Subqueries and JOINs often express the same logic. Rewriting a subquery as a JOIN can improve performance on large tables and give the query optimizer more flexibility. This guide covers the most common rewrites.

## When to consider rewriting

- The subquery is correlated and runs once per outer row on a large table.
- `EXPLAIN` shows a full table scan inside the subquery.
- You need to use the result of the subquery in multiple places in `SELECT`.
- The subquery limits optimizer strategies (some older MySQL versions treat correlated subqueries less efficiently than joins).

## Pattern 1: IN subquery to INNER JOIN

```sql
-- Subquery
SELECT name
FROM employees
WHERE department_id IN (
    SELECT department_id FROM departments WHERE location = 'London'
);

-- JOIN equivalent
SELECT DISTINCT e.name
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id
WHERE d.location = 'London';
```

Note: `DISTINCT` is needed when the join could produce duplicate rows (one employee row per matching department row). If `department_id` is a primary key in `departments`, there is at most one match per employee, so `DISTINCT` is optional.

## Pattern 2: NOT IN to LEFT JOIN anti-join

```sql
-- Subquery (unsafe with NULLs)
SELECT name
FROM employees
WHERE department_id NOT IN (
    SELECT department_id FROM departments WHERE location = 'London'
);

-- LEFT JOIN anti-join (safe with NULLs)
SELECT e.name
FROM employees e
LEFT JOIN departments d
    ON e.department_id = d.department_id
    AND d.location = 'London'
WHERE d.department_id IS NULL;
```

The LEFT JOIN anti-join is safer because `NOT IN` returns no rows if the subquery contains any `NULL` values.

## Pattern 3: EXISTS to INNER JOIN (semi-join)

```sql
-- EXISTS subquery
SELECT e.name
FROM employees e
WHERE EXISTS (
    SELECT 1 FROM expense_reports er WHERE er.employee_id = e.employee_id
);

-- JOIN with DISTINCT
SELECT DISTINCT e.name
FROM employees e
INNER JOIN expense_reports er ON e.employee_id = er.employee_id;
```

MySQL's optimizer often rewrites `EXISTS` to a semi-join internally, so the performance may already be equivalent.

## Pattern 4: NOT EXISTS to LEFT JOIN anti-join

```sql
-- NOT EXISTS
SELECT e.name
FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM expense_reports er WHERE er.employee_id = e.employee_id
);

-- LEFT JOIN anti-join
SELECT e.name
FROM employees e
LEFT JOIN expense_reports er ON e.employee_id = er.employee_id
WHERE er.employee_id IS NULL;
```

## Pattern 5: Correlated scalar subquery in SELECT to LEFT JOIN

```sql
-- Correlated scalar subquery (runs once per row)
SELECT
    c.name,
    (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id) AS order_count
FROM customers c;

-- LEFT JOIN + GROUP BY (usually one pass over orders)
SELECT
    c.name,
    COUNT(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name;
```

## Pattern 6: Derived table subquery to JOIN

```sql
-- Derived table
SELECT c.name, rev.total
FROM customers c
INNER JOIN (
    SELECT customer_id, SUM(total) AS total
    FROM orders
    GROUP BY customer_id
) AS rev ON c.customer_id = rev.customer_id;

-- This is already a JOIN. The derived table form pre-aggregates, which is often desirable.
-- You could also write it without the derived table:
SELECT c.name, SUM(o.total) AS total
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name;
```

Both versions are equivalent. The pre-aggregated derived table can be faster when the aggregation reduces rows significantly before the join.

## When NOT to rewrite

- `NOT IN` with a small, static list is readable and fast.
- `EXISTS` / `NOT EXISTS` with proper indexes often performs identically to a JOIN.
- A correlated subquery in `SELECT` is sometimes clearer than a `LEFT JOIN` + `GROUP BY`.
- MySQL 8 already rewrites many subqueries to semi-joins automatically.

## Verifying with EXPLAIN

Compare execution plans before and after rewriting:

```sql
EXPLAIN
SELECT name FROM employees
WHERE department_id IN (SELECT department_id FROM departments WHERE location = 'London');

EXPLAIN
SELECT DISTINCT e.name FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id
WHERE d.location = 'London';
```

Check the `type`, `key`, and `rows` columns. If both plans are identical, the optimizer already handled the rewrite.

## Summary

The most common subquery-to-JOIN rewrites are: `IN` to `INNER JOIN DISTINCT`, `NOT IN` to `LEFT JOIN ... IS NULL`, and correlated SELECT-list subqueries to `LEFT JOIN + GROUP BY`. Always use `EXPLAIN` to verify that the rewrite actually improves the plan, since MySQL 8 already applies many of these transformations automatically.
