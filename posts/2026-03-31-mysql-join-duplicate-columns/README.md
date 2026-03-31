# How to Handle Duplicate Columns in MySQL Joins

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Join, Database, Query

Description: Learn how to handle duplicate column names that appear when joining MySQL tables, using column aliases, USING, and explicit SELECT lists to produce clean result sets.

---

When you join two tables that share column names, such as `id`, `name`, or `created_at`, the result set can contain duplicate column names. This causes confusion in application code, ORM mappings, and `SELECT *` output. This guide shows how to deal with it cleanly.

## The problem

```sql
CREATE TABLE employees (
    id         INT PRIMARY KEY,
    name       VARCHAR(100),
    dept_id    INT,
    created_at DATETIME
);

CREATE TABLE departments (
    id         INT PRIMARY KEY,
    name       VARCHAR(60),
    created_at DATETIME
);

-- Both tables have id, name, created_at
SELECT *
FROM employees e
INNER JOIN departments d ON e.dept_id = d.id;
```

The result contains columns: `id`, `name`, `dept_id`, `created_at`, `id`, `name`, `created_at`. Applications that fetch columns by name may silently use the wrong value.

## Solution 1: Explicit column list with aliases

The safest approach is always to name the columns you need:

```sql
SELECT
    e.id         AS employee_id,
    e.name       AS employee_name,
    e.created_at AS employee_created,
    d.id         AS department_id,
    d.name       AS department_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.id;
```

Every column in the result set now has a unique, descriptive name.

## Solution 2: Use USING to collapse the join column

When the join column has the same name in both tables, `USING` merges the two copies into one:

```sql
-- If both tables have an 'id' column that represents the same concept (unusual)
-- More realistic: join on a shared foreign key column
CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT
);

CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name        VARCHAR(100)
);

-- USING collapses customer_id into one column
SELECT customer_id, c.name, o.order_id
FROM customers c
INNER JOIN orders o USING (customer_id);
```

`USING` does not help for non-join columns that happen to share a name (`name`, `created_at`). You must alias those manually.

## Solution 3: Qualify columns with table prefix

Instead of aliases, qualify every ambiguous column with its table alias:

```sql
SELECT
    e.id,
    e.name,
    d.id   AS dept_id,
    d.name AS dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.id;
```

Note that the column label in the result set is still `id` and `dept_id`. Client applications see the last unaliased column name, which can still cause collisions if you fetch by column name rather than position.

## Impact on application code

Most MySQL connectors return rows as dictionaries. When two columns share the same name, behaviour varies:

| Connector | Duplicate column behaviour |
|---|---|
| mysql2 (Node.js) | Second column overwrites first in object mode |
| Go `database/sql` | Accessible by index; `Columns()` returns both names |
| Python mysql-connector | Returns list of tuples; column names via `description` |
| PHP PDO fetchAssoc | Second column overwrites first |

Always use explicit aliases when the application fetches columns by name.

## Detecting duplicates before they cause bugs

```sql
-- Check for shared column names between two tables
SELECT c.COLUMN_NAME
FROM information_schema.COLUMNS c
WHERE c.TABLE_SCHEMA = 'your_db'
  AND c.TABLE_NAME IN ('employees', 'departments')
GROUP BY c.COLUMN_NAME
HAVING COUNT(*) > 1;
```

## Practical example: clean multi-table join

```sql
SELECT
    e.id          AS employee_id,
    e.name        AS employee_name,
    e.created_at  AS hired_at,
    d.id          AS department_id,
    d.name        AS department_name,
    d.created_at  AS dept_founded
FROM employees e
INNER JOIN departments d ON e.dept_id = d.id
ORDER BY e.name;
```

Every result column is unambiguous regardless of what the client application does.

## Summary

Duplicate columns in MySQL joins arise from `SELECT *` across tables with shared column names. The cleanest solution is an explicit `SELECT` list with column aliases. Use `USING` to collapse the join column into a single copy, and query `information_schema.COLUMNS` to proactively identify name collisions between tables you plan to join.
