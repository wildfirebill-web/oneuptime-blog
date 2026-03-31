# How to Use the COMMENT Clause for Tables and Columns in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, DDL, Table, Column, Documentation

Description: Learn how to add, update, and view comments on MySQL tables and columns to document schema intent and make your database self-describing.

---

## Why Use Comments in MySQL?

MySQL allows you to attach descriptive text to tables and individual columns using the `COMMENT` clause. These comments are stored in the information schema and are visible in tools like MySQL Workbench, phpMyAdmin, and `SHOW CREATE TABLE`. Well-placed comments act as inline documentation, reducing the cognitive overhead of understanding an unfamiliar schema.

## Adding a Comment When Creating a Table

```sql
CREATE TABLE orders (
    id         INT          NOT NULL AUTO_INCREMENT PRIMARY KEY COMMENT 'Surrogate primary key',
    customer_id INT         NOT NULL COMMENT 'References customers.id',
    status     VARCHAR(20)  NOT NULL DEFAULT 'pending' COMMENT 'Order lifecycle state: pending, shipped, delivered, cancelled',
    total      DECIMAL(12,2) NOT NULL COMMENT 'Total order value in USD',
    created_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT 'UTC creation timestamp'
) ENGINE = InnoDB COMMENT = 'Customer purchase orders';
```

## Adding a Comment to an Existing Column

```sql
ALTER TABLE orders
    MODIFY COLUMN status VARCHAR(20) NOT NULL DEFAULT 'pending'
    COMMENT 'Order lifecycle state: pending, shipped, delivered, cancelled';
```

Note that `MODIFY COLUMN` requires the full column definition - you cannot set only the comment.

## Adding a Comment to an Existing Table

```sql
ALTER TABLE orders COMMENT = 'Customer purchase orders - see wiki/orders for business rules';
```

## Viewing Table and Column Comments

### Using SHOW CREATE TABLE

```sql
SHOW CREATE TABLE orders\G
```

```text
CREATE TABLE `orders` (
  `id` int NOT NULL AUTO_INCREMENT COMMENT 'Surrogate primary key',
  ...
) ENGINE=InnoDB COMMENT='Customer purchase orders'
```

### Querying the Information Schema

```sql
-- Table comments
SELECT TABLE_NAME, TABLE_COMMENT
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database';

-- Column comments
SELECT COLUMN_NAME, COLUMN_COMMENT
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'your_database'
  AND TABLE_NAME = 'orders'
ORDER BY ORDINAL_POSITION;
```

## Removing a Comment

Set the comment to an empty string:

```sql
ALTER TABLE orders COMMENT = '';

ALTER TABLE orders
    MODIFY COLUMN status VARCHAR(20) NOT NULL DEFAULT 'pending' COMMENT '';
```

## Comment Length Limits

- Table comments: up to 2048 characters
- Column comments: up to 1024 characters

## Best Practices

```sql
-- Document enum-like columns with allowed values
ALTER TABLE users
    MODIFY COLUMN role TINYINT NOT NULL DEFAULT 1
    COMMENT 'User role: 1=viewer, 2=editor, 3=admin';

-- Note deprecation in comments instead of dropping columns immediately
ALTER TABLE legacy_reports
    MODIFY COLUMN old_format_flag TINYINT NULL
    COMMENT 'DEPRECATED: no longer populated as of 2025-01 migration. Remove after Q2 cleanup.';
```

## Summary

MySQL's `COMMENT` clause lets you embed documentation directly in the schema. Use column comments to describe allowed values, foreign key targets, and units. Use table comments to link to external documentation or describe the business purpose. Query `information_schema.COLUMNS` and `information_schema.TABLES` to retrieve comments programmatically.
