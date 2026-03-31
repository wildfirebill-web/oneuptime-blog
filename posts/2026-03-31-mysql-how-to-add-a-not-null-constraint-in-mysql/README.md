# How to Add a NOT NULL Constraint in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Ddl, Constraints, Schema Design

Description: Learn how to add a NOT NULL constraint to an existing MySQL column using ALTER TABLE MODIFY, including handling existing NULL values safely.

---

## What Is a NOT NULL Constraint?

A `NOT NULL` constraint prevents a column from storing NULL values. Any attempt to insert or update a row without providing a value for a `NOT NULL` column (and without a default) results in an error.

## Adding NOT NULL with ALTER TABLE MODIFY

In MySQL, `NOT NULL` is part of the column definition. To add it to an existing column, use `MODIFY COLUMN`:

```sql
ALTER TABLE table_name
MODIFY COLUMN column_name datatype NOT NULL;
```

## Example

```sql
-- Before: email allows NULL
ALTER TABLE users
MODIFY COLUMN email VARCHAR(100) NOT NULL;
```

You must specify the complete data type when using `MODIFY COLUMN`.

## Step 1: Check for NULL Values First

Before adding `NOT NULL`, check whether any existing rows have NULL in the column:

```sql
SELECT COUNT(*) AS null_count
FROM users
WHERE email IS NULL;
```

If `null_count > 0`, MySQL will reject the `ALTER TABLE`. You must first fill in those NULLs.

## Step 2: Fill in Existing NULLs

```sql
-- Option 1: update with a placeholder value
UPDATE users
SET email = CONCAT('unknown_', id, '@example.com')
WHERE email IS NULL;

-- Option 2: delete rows with NULL (if appropriate)
DELETE FROM users WHERE email IS NULL;

-- Option 3: set a default
UPDATE users SET email = 'noreply@example.com' WHERE email IS NULL;
```

## Step 3: Add the NOT NULL Constraint

```sql
ALTER TABLE users
MODIFY COLUMN email VARCHAR(100) NOT NULL;
```

## Adding NOT NULL with a DEFAULT

Providing a `DEFAULT` allows MySQL to automatically use the default for any NULLs during the schema change (this behavior depends on SQL mode):

```sql
ALTER TABLE users
MODIFY COLUMN is_active TINYINT(1) NOT NULL DEFAULT 1;
```

In strict SQL mode (`STRICT_TRANS_TABLES`), existing NULL values still cause an error unless updated first.

## Removing NOT NULL (Making Nullable)

```sql
ALTER TABLE users
MODIFY COLUMN phone VARCHAR(20) NULL;
```

## Adding NOT NULL at Table Creation

The best practice is to define `NOT NULL` when creating the table:

```sql
CREATE TABLE users (
    id       INT          NOT NULL AUTO_INCREMENT,
    username VARCHAR(50)  NOT NULL,
    email    VARCHAR(100) NOT NULL,
    phone    VARCHAR(20),          -- nullable
    PRIMARY KEY (id)
);
```

## Checking Column Nullability

```sql
SHOW COLUMNS FROM users;

-- Or via INFORMATION_SCHEMA
SELECT
    COLUMN_NAME,
    IS_NULLABLE,
    COLUMN_DEFAULT
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = DATABASE()
  AND TABLE_NAME = 'users';
```

Output shows `NO` in `IS_NULLABLE` for `NOT NULL` columns.

## NOT NULL vs Default Values

```text
NOT NULL alone:  must provide a value on every INSERT (no default fallback)
NOT NULL + DEFAULT: uses default when no value is provided
NULL (no constraint): allows NULL; NULL is stored if no value given and no default
```

## Summary

To add a `NOT NULL` constraint to an existing MySQL column, first check for and resolve any NULL values in that column, then use `ALTER TABLE ... MODIFY COLUMN column_name datatype NOT NULL`. Include a `DEFAULT` value when you want MySQL to use it for inserts that omit the column. Always verify nullability with `SHOW COLUMNS` or `INFORMATION_SCHEMA.COLUMNS` before and after the alteration.
