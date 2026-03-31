# How to Change a Column Data Type in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, DDL, ALTER TABLE, Schema Design

Description: Learn how to change a column's data type in MySQL using ALTER TABLE MODIFY or CHANGE, with considerations for data safety and online DDL.

---

## Changing a Column Data Type with MODIFY

```sql
ALTER TABLE table_name
MODIFY COLUMN column_name new_datatype [constraints];
```

`MODIFY COLUMN` changes the data type and constraints of an existing column while keeping the same name. `COLUMN` is optional but recommended for clarity.

## Simple Data Type Change

```sql
-- Change from INT to BIGINT
ALTER TABLE orders
MODIFY COLUMN user_id BIGINT NOT NULL;

-- Change VARCHAR length
ALTER TABLE users
MODIFY COLUMN username VARCHAR(100) NOT NULL;

-- Change from VARCHAR to TEXT
ALTER TABLE articles
MODIFY COLUMN body TEXT;
```

## Using CHANGE for Rename + Type Change

`CHANGE` allows you to rename and retype in one step:

```sql
ALTER TABLE table_name
CHANGE old_column_name new_column_name new_datatype [constraints];
```

```sql
ALTER TABLE users
CHANGE user_phone phone_number VARCHAR(20) NOT NULL DEFAULT '';
```

## Checking the Current Column Definition

Before modifying, check the current definition to avoid accidental changes:

```sql
SHOW COLUMNS FROM users LIKE 'username';
```

```text
Field    | Type        | Null | Key | Default | Extra
---------+-------------+------+-----+---------+-------
username | varchar(50) | NO   | UNI | NULL    |
```

## Data Type Widening vs Narrowing

Widening (e.g., INT to BIGINT, VARCHAR(50) to VARCHAR(100)) is safe and does not risk data loss:

```sql
ALTER TABLE users
MODIFY COLUMN username VARCHAR(200) NOT NULL;
```

Narrowing (e.g., VARCHAR(200) to VARCHAR(50)) can truncate data. MySQL will warn but may proceed:

```sql
-- Check maximum length before narrowing
SELECT MAX(LENGTH(username)) FROM users;

-- Then narrow if safe
ALTER TABLE users
MODIFY COLUMN username VARCHAR(50) NOT NULL;
```

## Changing Numeric Precision

```sql
ALTER TABLE products
MODIFY COLUMN price DECIMAL(12, 2) NOT NULL DEFAULT 0.00;
```

## Changing to an ENUM

```sql
ALTER TABLE orders
MODIFY COLUMN status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled')
    NOT NULL DEFAULT 'pending';
```

Existing values not in the ENUM list will be converted to an empty string (if the column allows it) or cause an error.

## Adding or Removing NOT NULL with MODIFY

```sql
-- Add NOT NULL constraint (set a default first for existing NULL rows)
UPDATE users SET preferences = '{}' WHERE preferences IS NULL;
ALTER TABLE users
MODIFY COLUMN preferences JSON NOT NULL;

-- Remove NOT NULL
ALTER TABLE users
MODIFY COLUMN phone VARCHAR(20) NULL;
```

## Online DDL for Data Type Changes

Some type changes are instant in MySQL 8.0 with InnoDB:

```sql
-- VARCHAR widening is instant (within same length threshold)
ALTER TABLE users
MODIFY COLUMN username VARCHAR(200) NOT NULL,
ALGORITHM=INSTANT;

-- Other changes may require INPLACE or COPY
ALTER TABLE products
MODIFY COLUMN price DECIMAL(12, 4) NOT NULL,
ALGORITHM=INPLACE, LOCK=NONE;
```

If `ALGORITHM=INSTANT` is not supported, MySQL returns an error and you can fall back to `INPLACE` or `COPY`.

## Changing to JSON Type

```sql
-- First ensure existing data is valid JSON or update it
ALTER TABLE users
MODIFY COLUMN settings JSON;
```

MySQL validates JSON on insert/update after the type change.

## Summary

Use `ALTER TABLE ... MODIFY COLUMN` to change a column's data type in MySQL. Always check the current definition with `SHOW COLUMNS` before modifying, and verify existing data is compatible with the new type - especially when narrowing or changing to more restrictive types. In MySQL 8.0 with InnoDB, use `ALGORITHM=INSTANT` or `ALGORITHM=INPLACE, LOCK=NONE` to minimize disruption on production tables.
