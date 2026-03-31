# How to Validate Data Integrity in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Data Integrity, Constraint, Validation, Foreign Key

Description: Learn how to validate data integrity in MySQL by checking referential integrity, constraint violations, data ranges, and orphaned records before and after migrations.

---

## Why Data Integrity Validation Matters

Data integrity problems - orphaned foreign keys, out-of-range values, and constraint violations - cause application errors that are hard to trace. Proactively validating integrity catches problems early, especially after bulk imports or migrations.

## Checking for Orphaned Foreign Key Records

Find rows in child tables that reference non-existent parent records:

```sql
-- Find orders without a valid customer
SELECT o.id, o.customer_id
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id
WHERE c.id IS NULL;

-- Find order_items without a valid order
SELECT oi.id, oi.order_id
FROM order_items oi
LEFT JOIN orders o ON oi.order_id = o.id
WHERE o.id IS NULL;
```

## Validating NOT NULL Constraints

Check for NULL values in columns that should never be null:

```sql
SELECT
  SUM(CASE WHEN customer_id IS NULL THEN 1 ELSE 0 END) AS null_customer_id,
  SUM(CASE WHEN total_amount IS NULL THEN 1 ELSE 0 END) AS null_total,
  SUM(CASE WHEN status IS NULL THEN 1 ELSE 0 END)       AS null_status
FROM orders;
```

## Checking Data Range Constraints

Validate that numeric and date values fall within acceptable ranges:

```sql
-- Prices should never be negative
SELECT id, price FROM products WHERE price < 0;

-- Quantities should be positive
SELECT id, quantity FROM order_items WHERE quantity <= 0;

-- Dates should be in a reasonable range
SELECT id, order_date FROM orders
WHERE order_date < '2000-01-01' OR order_date > NOW();
```

## Verifying Enum and Status Values

Check that categorical fields only contain allowed values:

```sql
SELECT DISTINCT status FROM orders
WHERE status NOT IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled');
```

## Checking Email Format Validity

Use a regex pattern to find malformed email addresses:

```sql
SELECT id, email
FROM customers
WHERE email NOT REGEXP '^[A-Za-z0-9._%+\\-]+@[A-Za-z0-9.\\-]+\\.[A-Za-z]{2,}$';
```

## Verifying Referential Integrity with Foreign Keys Disabled

After a bulk load with foreign key checks disabled, re-validate manually:

```sql
-- Temporarily check without enforcement
SET foreign_key_checks = 0;
-- ... bulk load ...
SET foreign_key_checks = 1;

-- Now validate manually
SELECT COUNT(*) AS orphaned_items
FROM order_items oi
LEFT JOIN orders o ON oi.order_id = o.id
WHERE o.id IS NULL;
```

## Running an Integrity Report

Create a summary report of all integrity issues:

```sql
SELECT 'orphaned_order_items' AS check_name,
       COUNT(*) AS violations
FROM order_items oi
LEFT JOIN orders o ON oi.order_id = o.id
WHERE o.id IS NULL

UNION ALL

SELECT 'negative_prices',
       COUNT(*)
FROM products WHERE price < 0

UNION ALL

SELECT 'invalid_email',
       COUNT(*)
FROM customers
WHERE email NOT REGEXP '^[^@]+@[^@]+\\.[^@]+$';
```

## Summary

Validate MySQL data integrity by checking for orphaned foreign keys with `LEFT JOIN ... WHERE parent.id IS NULL`, scanning for NULL violations, range errors, and invalid enum values. Run a consolidated report using `UNION ALL` to get a complete picture of data quality issues. Re-enable foreign key constraints and address all violations before deploying to production.
