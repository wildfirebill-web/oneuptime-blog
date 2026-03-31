# How to Create a View in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, View, DDL, Query, Database Object

Description: Learn how to create a MySQL view using CREATE VIEW, understand how views work, and see practical examples for simplifying complex queries.

---

A MySQL view is a named query stored in the database. It behaves like a virtual table - you can query it with `SELECT`, and MySQL executes the underlying query each time. Views simplify complex SQL, enforce column-level security, and present a stable interface to applications even when the underlying schema changes.

## Basic CREATE VIEW Syntax

```sql
CREATE VIEW view_name AS
SELECT column1, column2, ...
FROM table_name
WHERE condition;
```

## Example 1 - Simple Column Projection

Create a view that exposes only non-sensitive customer columns:

```sql
CREATE VIEW customer_public AS
SELECT customer_id, first_name, last_name, city, country
FROM customers;
```

Query it like a table:

```sql
SELECT * FROM customer_public WHERE country = 'US';
```

## Example 2 - Joining Multiple Tables

A view can encapsulate a multi-table join so application code does not need to repeat it:

```sql
CREATE VIEW order_details_view AS
SELECT
    o.order_id,
    o.created_at,
    c.first_name,
    c.last_name,
    p.name        AS product_name,
    oi.quantity,
    oi.unit_price,
    oi.quantity * oi.unit_price AS line_total
FROM orders o
JOIN customers c  ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p   ON oi.product_id = p.product_id;
```

Application queries become clean:

```sql
SELECT order_id, first_name, last_name, line_total
FROM order_details_view
WHERE created_at >= '2026-01-01';
```

## Example 3 - Aggregated View

Views can contain aggregate functions to pre-define reporting queries:

```sql
CREATE VIEW monthly_revenue AS
SELECT
    DATE_FORMAT(created_at, '%Y-%m') AS month,
    COUNT(*)                          AS order_count,
    SUM(total)                        AS revenue
FROM orders
GROUP BY DATE_FORMAT(created_at, '%Y-%m');
```

## View Security with DEFINER and INVOKER

By default a view uses `SQL SECURITY DEFINER` - it runs with the privileges of the user who created it. Use `SQL SECURITY INVOKER` to run with the privileges of the user executing the query:

```sql
CREATE DEFINER = 'dba'@'localhost'
SQL SECURITY INVOKER
VIEW customer_public AS
SELECT customer_id, first_name, last_name FROM customers;
```

## Inspect a View's Definition

```sql
SHOW CREATE VIEW order_details_view\G

SELECT VIEW_DEFINITION
FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'order_details_view';
```

## Summary

Create a MySQL view with `CREATE VIEW name AS SELECT ...` to encapsulate complex joins, aggregations, or column filters behind a simple name. Views improve query readability, enforce data access rules, and decouple application SQL from underlying table structures.
