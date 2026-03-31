# How to Use CREATE VIEW Statement in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, View, SQL, Database Design

Description: Learn how to use the CREATE VIEW statement in MySQL to define reusable virtual tables that simplify complex queries and enforce security.

---

## What Is a MySQL View

A view is a named query stored in the database that behaves like a virtual table. When you query a view, MySQL executes the underlying SELECT statement and returns the result. Views do not store data themselves (unless you use materialized-view patterns) but provide a stable interface for complex queries.

Common use cases include simplifying joins, hiding sensitive columns, enforcing row-level security, and providing backward-compatible table interfaces after schema changes.

## Basic Syntax

```sql
CREATE [OR REPLACE] VIEW view_name [(column_list)]
AS select_statement
[WITH [CASCADED | LOCAL] CHECK OPTION];
```

## Creating a Simple View

```sql
CREATE VIEW active_users AS
    SELECT id, username, email, created_at
    FROM users
    WHERE status = 'active';
```

Query the view like a table:

```sql
SELECT * FROM active_users WHERE email LIKE '%@example.com';
```

## Creating a View with Renamed Columns

Rename columns inline using the column list:

```sql
CREATE VIEW order_summary (order_id, customer, total_items, order_date)
AS
    SELECT o.id, CONCAT(u.first_name, ' ', u.last_name), COUNT(i.id), o.created_at
    FROM orders o
    JOIN users u ON u.id = o.user_id
    JOIN order_items i ON i.order_id = o.id
    GROUP BY o.id, u.first_name, u.last_name, o.created_at;
```

## Replacing an Existing View

Use `CREATE OR REPLACE` to update an existing view without dropping it first:

```sql
CREATE OR REPLACE VIEW active_users AS
    SELECT id, username, email, created_at, last_login
    FROM users
    WHERE status = 'active';
```

## Updatable Views

A view is updatable if MySQL can map it back to a single base table with no aggregates, DISTINCT, GROUP BY, HAVING, UNION, or subqueries. INSERT/UPDATE/DELETE through the view are then possible:

```sql
CREATE VIEW user_emails AS
    SELECT id, email FROM users;

-- Update through the view
UPDATE user_emails SET email = 'new@example.com' WHERE id = 5;
```

## WITH CHECK OPTION

`WITH CHECK OPTION` prevents inserting or updating rows through a view that would not satisfy the view's WHERE clause:

```sql
CREATE VIEW active_users AS
    SELECT id, username, status
    FROM users
    WHERE status = 'active'
WITH CHECK OPTION;

-- This will fail because status 'inactive' violates the view's filter
UPDATE active_users SET status = 'inactive' WHERE id = 5;
-- ERROR 1369: CHECK OPTION failed
```

## Security Views - Hiding Sensitive Columns

Views are often used to expose only safe columns to application users:

```sql
CREATE VIEW public_employees AS
    SELECT id, first_name, last_name, department, job_title
    FROM employees;
-- salary, ssn, and personal_email are excluded
```

Grant access only to the view, not the base table:

```sql
GRANT SELECT ON mydb.public_employees TO 'app_reader'@'%';
```

## Inspecting and Dropping Views

```sql
-- List views in the current database
SHOW FULL TABLES WHERE table_type = 'VIEW';

-- Show the view definition
SHOW CREATE VIEW active_users\G

-- Query from information_schema
SELECT table_name, view_definition
FROM information_schema.views
WHERE table_schema = 'mydb';

-- Drop a view
DROP VIEW IF EXISTS active_users;
```

## Performance Considerations

Views do not cache results. Each query against a view re-executes the underlying SELECT. For complex views with heavy aggregation, this can be expensive. Options:

- Use the `MERGE` algorithm (default for simple views) for MySQL to inline the view query
- Use the `TEMPTABLE` algorithm for views with aggregates (MySQL creates a temp table)
- Consider a scheduled batch process that writes results to a real table for very expensive views

```sql
CREATE ALGORITHM=MERGE VIEW fast_view AS
    SELECT id, name FROM products WHERE active = 1;
```

## Summary

The `CREATE VIEW` statement provides a powerful abstraction layer in MySQL for simplifying complex queries, enforcing column-level security, and creating stable APIs over changing schemas. Use `WITH CHECK OPTION` to maintain data integrity on updatable views, and choose the appropriate algorithm for performance-sensitive scenarios.
