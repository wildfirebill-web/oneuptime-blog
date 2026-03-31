# How to Add a Column to an Existing Table in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Ddl, Alter Table, Schema Design

Description: Learn how to add columns to existing MySQL tables using ALTER TABLE with options for position, defaults, and online DDL for minimal downtime.

---

## Basic ADD COLUMN Syntax

```sql
ALTER TABLE table_name
ADD COLUMN column_name datatype [constraints];
```

## Adding a Simple Column

```sql
ALTER TABLE users
ADD COLUMN phone VARCHAR(20);
```

By default, new columns are appended to the end of the table.

## Adding a Column with a Default Value

```sql
ALTER TABLE users
ADD COLUMN is_active BOOLEAN NOT NULL DEFAULT TRUE;
```

The default value is applied to all existing rows immediately.

## Adding a Column with NOT NULL (No Default)

```sql
ALTER TABLE users
ADD COLUMN full_name VARCHAR(100) NOT NULL;
```

For existing rows, MySQL will set the column to empty string for string types, 0 for numeric types, etc. If you need a specific value, provide a DEFAULT:

```sql
ALTER TABLE users
ADD COLUMN full_name VARCHAR(100) NOT NULL DEFAULT '';
```

## Specifying Column Position

Use `FIRST` to add at the beginning, or `AFTER column_name` to insert after a specific column:

```sql
-- Add as first column
ALTER TABLE users
ADD COLUMN uuid CHAR(36) NOT NULL FIRST;

-- Add after a specific column
ALTER TABLE users
ADD COLUMN last_login DATETIME AFTER email;
```

## Adding Multiple Columns at Once

```sql
ALTER TABLE users
ADD COLUMN phone       VARCHAR(20)   AFTER email,
ADD COLUMN address     VARCHAR(255),
ADD COLUMN birth_date  DATE;
```

Combining multiple `ADD COLUMN` operations into a single `ALTER TABLE` is more efficient than running separate statements.

## Adding a Column with AUTO_INCREMENT

`AUTO_INCREMENT` can only be added to integer columns that are part of a key:

```sql
-- For a new primary key column (if no pk exists)
ALTER TABLE logs
ADD COLUMN id INT NOT NULL AUTO_INCREMENT PRIMARY KEY FIRST;
```

## Adding a JSON Column

```sql
ALTER TABLE products
ADD COLUMN metadata JSON;
```

## Adding a Generated (Computed) Column

MySQL supports virtual and stored generated columns:

```sql
-- Virtual: computed on read, not stored on disk
ALTER TABLE orders
ADD COLUMN total_with_tax DECIMAL(10,2)
    GENERATED ALWAYS AS (total * 1.1) VIRTUAL;

-- Stored: computed and stored on disk
ALTER TABLE orders
ADD COLUMN total_with_tax DECIMAL(10,2)
    GENERATED ALWAYS AS (total * 1.1) STORED;
```

## Online DDL (Avoiding Table Lock)

In MySQL 8.0 with InnoDB, many `ALTER TABLE` operations are online and do not lock the table for reads/writes. You can specify the algorithm explicitly:

```sql
ALTER TABLE users
ADD COLUMN preferences JSON,
ALGORITHM=INPLACE, LOCK=NONE;
```

- `ALGORITHM=INPLACE` avoids a full table rebuild when possible.
- `LOCK=NONE` allows concurrent reads and writes during the DDL.

If `ALGORITHM=INPLACE` is not supported for the operation, MySQL returns an error so you can decide how to proceed.

## Checking the Table Structure After Alteration

```sql
DESCRIBE users;

SHOW COLUMNS FROM users;
```

## Summary

`ALTER TABLE ... ADD COLUMN` adds new columns to an existing MySQL table. Specify `DEFAULT` values for instant population of existing rows, control position with `FIRST` or `AFTER`, and combine multiple column additions into a single statement for efficiency. In MySQL 8.0 with InnoDB, use `ALGORITHM=INPLACE, LOCK=NONE` for online DDL to minimize impact on production tables.
