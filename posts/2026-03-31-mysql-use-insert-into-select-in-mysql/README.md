# How to Use INSERT INTO ... SELECT in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Insert Into Select, Data Migration, Sql, Database

Description: Learn how to use INSERT INTO ... SELECT in MySQL to copy rows from one table to another, with filtering, transformation, and bulk insert techniques.

---

## Introduction

`INSERT INTO ... SELECT` in MySQL allows you to insert rows into a table using the results of a `SELECT` query instead of specifying literal values. This is useful for copying data between tables, archiving records, populating summary tables, and bulk data migration.

## Basic Syntax

```sql
INSERT INTO target_table (column1, column2, ...)
SELECT column1, column2, ...
FROM source_table
WHERE condition;
```

## Simple Copy Between Tables

Copy all rows from one table to another with identical structure:

```sql
INSERT INTO employees_backup
SELECT * FROM employees;
```

## Copy with Column Mapping

When column names differ, specify them explicitly:

```sql
INSERT INTO archive_orders (order_id, customer_name, total, archived_at)
SELECT id, customer, amount, NOW()
FROM orders
WHERE status = 'completed';
```

## Copying with Transformation

You can apply functions and expressions during the copy:

```sql
INSERT INTO user_summaries (user_id, full_name, email_lower, created_year)
SELECT
  id,
  CONCAT(first_name, ' ', last_name),
  LOWER(email),
  YEAR(created_at)
FROM users;
```

## Insert with Aggregation

Populate a summary table from aggregated data:

```sql
INSERT INTO monthly_sales_summary (year, month, total_revenue, order_count)
SELECT
  YEAR(order_date),
  MONTH(order_date),
  SUM(total_amount),
  COUNT(*)
FROM orders
WHERE status = 'completed'
GROUP BY YEAR(order_date), MONTH(order_date);
```

## Insert from a JOIN

You can SELECT from a JOIN and INSERT the result:

```sql
INSERT INTO order_details_flat (order_id, customer_name, product_name, quantity)
SELECT
  o.id,
  c.name,
  p.name,
  oi.quantity
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id;
```

## Archiving Old Data

Move old records to an archive table before deletion:

```sql
-- First, copy to archive
INSERT INTO orders_archive
SELECT * FROM orders
WHERE order_date < DATE_SUB(NOW(), INTERVAL 2 YEAR);

-- Then delete originals
DELETE FROM orders
WHERE order_date < DATE_SUB(NOW(), INTERVAL 2 YEAR);
```

## Avoid Duplicates with IGNORE

If the target table has a unique constraint and duplicates might exist, use `INSERT IGNORE`:

```sql
INSERT IGNORE INTO product_catalog_new
SELECT id, sku, name, price
FROM product_catalog_old;
```

## INSERT INTO ... SELECT with LIMIT

You can use LIMIT in the SELECT to control how many rows are inserted per batch:

```sql
INSERT INTO archive_logs
SELECT * FROM logs
WHERE log_date < '2023-01-01'
ORDER BY log_date ASC
LIMIT 10000;
```

This is useful for batching large migrations.

## INSERT INTO ... SELECT in a Transaction

Wrap in a transaction for safety when also deleting:

```sql
START TRANSACTION;

INSERT INTO orders_archive
SELECT * FROM orders WHERE status = 'cancelled' AND order_date < '2023-01-01';

DELETE FROM orders WHERE status = 'cancelled' AND order_date < '2023-01-01';

COMMIT;
```

## Creating a Table and Inserting in One Step

Use `CREATE TABLE ... SELECT` to create and populate in one command:

```sql
CREATE TABLE new_customers AS
SELECT id, name, email, created_at
FROM customers
WHERE created_at >= '2024-01-01';
```

Note: This does not copy indexes or constraints - add them manually afterward.

## Performance Tips

- For large inserts, disable indexes temporarily or use `INSERT DELAYED` behavior by batching.
- Index the target table columns after the bulk insert.
- Set `unique_checks = 0` and `foreign_key_checks = 0` during bulk loads (restore after):

```sql
SET foreign_key_checks = 0;
SET unique_checks = 0;

INSERT INTO large_target_table SELECT * FROM large_source_table;

SET unique_checks = 1;
SET foreign_key_checks = 1;
```

## Summary

`INSERT INTO ... SELECT` is a powerful way to copy, migrate, and transform data between tables in MySQL. It supports filtering with WHERE, transformations with expressions, aggregation with GROUP BY, and joining multiple tables. Use it for archiving, populating summary tables, and bulk data migrations.
