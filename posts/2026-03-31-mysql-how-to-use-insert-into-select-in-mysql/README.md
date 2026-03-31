# How to Use INSERT INTO ... SELECT in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INSERT INTO SELECT, Data Migration, SQL, Bulk Insert

Description: Learn how to use INSERT INTO ... SELECT in MySQL to copy, transform, and migrate data between tables in a single efficient statement.

---

## What Is INSERT INTO ... SELECT?

`INSERT INTO ... SELECT` copies rows from one table into another in a single SQL statement. It is more efficient than fetching data into the application and re-inserting it, and it supports transformations, filtering, and joining multiple source tables.

```sql
-- Copy all rows from source_table into target_table
INSERT INTO target_table (col1, col2, col3)
SELECT col1, col2, col3
FROM source_table;
```

## Basic Usage

```sql
-- Copy active customers to a separate archive table
CREATE TABLE active_customers_backup LIKE customers;

INSERT INTO active_customers_backup
SELECT * FROM customers WHERE is_active = 1;

-- Copy with column mapping (different column order)
INSERT INTO reports (report_date, sales_total, region)
SELECT sale_date, SUM(amount), region
FROM daily_sales
GROUP BY sale_date, region;
```

## Transforming Data During Insert

You can apply functions, expressions, and aggregations in the SELECT:

```sql
-- Copy with data transformations
INSERT INTO user_summary (user_id, full_name, email_lower, created_year)
SELECT
    id,
    CONCAT(first_name, ' ', last_name),
    LOWER(email),
    YEAR(created_at)
FROM users;

-- Calculate and store derived values
INSERT INTO product_stats (product_id, total_revenue, order_count, avg_order_value)
SELECT
    product_id,
    SUM(quantity * unit_price) AS total_revenue,
    COUNT(DISTINCT order_id) AS order_count,
    AVG(quantity * unit_price) AS avg_order_value
FROM order_items
GROUP BY product_id;
```

## Inserting from Multiple Tables with JOIN

```sql
-- Combine data from joined tables into a denormalized table
INSERT INTO order_details_flat (
    order_id, customer_name, product_name, quantity, unit_price, total
)
SELECT
    o.id,
    CONCAT(c.first_name, ' ', c.last_name),
    p.name,
    oi.quantity,
    oi.unit_price,
    oi.quantity * oi.unit_price
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON oi.product_id = p.id
WHERE o.status = 'completed';
```

## Conditional Filtering During Copy

```sql
-- Archive orders older than 1 year
INSERT INTO orders_archive
SELECT * FROM orders
WHERE order_date < DATE_SUB(CURDATE(), INTERVAL 1 YEAR);

-- After archiving, delete the original rows
DELETE FROM orders
WHERE order_date < DATE_SUB(CURDATE(), INTERVAL 1 YEAR);
```

## Using INSERT INTO ... SELECT with LIMIT

```sql
-- Migrate data in batches to avoid locking
INSERT INTO new_logs
SELECT * FROM old_logs
ORDER BY id
LIMIT 10000;
-- Repeat until all rows are migrated
```

## Populating a Lookup Table

```sql
-- Extract unique categories from a products table
INSERT IGNORE INTO categories (name)
SELECT DISTINCT category_name FROM products
WHERE category_name IS NOT NULL;
```

## Creating and Populating in One Step (CREATE TABLE ... SELECT)

```sql
-- Create a new table and populate it in one step
CREATE TABLE top_customers AS
SELECT
    c.id,
    c.name,
    c.email,
    SUM(o.amount) AS lifetime_value
FROM customers c
JOIN orders o ON o.customer_id = c.id
GROUP BY c.id
HAVING lifetime_value > 1000;
```

## Handling Duplicates During INSERT INTO SELECT

```sql
-- Skip rows that would violate unique constraints
INSERT IGNORE INTO email_list (email)
SELECT email FROM customers
WHERE email IS NOT NULL;

-- Or use ON DUPLICATE KEY UPDATE to upsert
INSERT INTO product_stats (product_id, total_revenue)
SELECT product_id, SUM(amount) FROM orders
GROUP BY product_id
ON DUPLICATE KEY UPDATE total_revenue = VALUES(total_revenue);
```

## Summary

`INSERT INTO ... SELECT` is a powerful and efficient pattern for copying, transforming, and migrating data between tables entirely within the database server. It supports all the power of a SELECT statement including JOINs, WHERE filters, aggregations, and expressions. Use it for data migration, archiving, building summary tables, and populating derived datasets without round-tripping data through the application layer.
