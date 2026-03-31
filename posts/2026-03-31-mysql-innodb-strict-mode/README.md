# How to Configure InnoDB Strict Mode in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Configuration, Strict Mode, Error Handling

Description: Learn how to configure InnoDB strict mode in MySQL to enforce stricter DDL validation and prevent silent data truncation or invalid table definitions.

---

InnoDB strict mode causes MySQL to return errors instead of warnings for certain DDL operations that create tables or indexes with invalid or problematic settings. It is analogous to SQL strict mode but specifically for InnoDB storage engine parameters.

## What InnoDB Strict Mode Controls

With `innodb_strict_mode=ON`, the following operations that would silently proceed with a warning instead throw an error:

- Creating a table with an invalid row format for the specified key block size
- Specifying `KEY_BLOCK_SIZE` incompatible with the row format
- Creating tables that exceed the row size limit for the page size
- Using `ROW_FORMAT=COMPRESSED` with a non-standard page size

```sql
-- Check current strict mode setting
SHOW VARIABLES LIKE 'innodb_strict_mode';
```

## Enabling InnoDB Strict Mode

```sql
-- Enable at session level
SET SESSION innodb_strict_mode = ON;

-- Enable globally (affects new connections)
SET GLOBAL innodb_strict_mode = ON;
```

Persist in `my.cnf` (ON by default since MySQL 5.7.7):

```text
[mysqld]
innodb_strict_mode=ON
```

## Observing the Difference

Without strict mode, an invalid `KEY_BLOCK_SIZE` generates a warning and MySQL uses a default value:

```sql
SET SESSION innodb_strict_mode = OFF;

-- This creates the table but ignores the invalid KEY_BLOCK_SIZE
CREATE TABLE test_table (
    id INT PRIMARY KEY,
    data VARCHAR(100)
) ROW_FORMAT=DYNAMIC KEY_BLOCK_SIZE=8;

SHOW WARNINGS;
-- Warning: InnoDB: ignoring KEY_BLOCK_SIZE=8 unless ROW_FORMAT=COMPRESSED
```

With strict mode enabled, the same statement raises an error:

```sql
SET SESSION innodb_strict_mode = ON;

CREATE TABLE test_table (
    id INT PRIMARY KEY,
    data VARCHAR(100)
) ROW_FORMAT=DYNAMIC KEY_BLOCK_SIZE=8;

-- ERROR 1005: Can't create table 'test_table'
-- KEY_BLOCK_SIZE requires ROW_FORMAT=COMPRESSED
```

## Row Size Validation

Strict mode enforces the InnoDB row size limit at table creation time rather than at data insertion time:

```sql
-- Without strict mode, this creates the table but may fail on insert
-- With strict mode, this fails immediately at CREATE TABLE
CREATE TABLE wide_table (
    col1 VARCHAR(300),
    col2 VARCHAR(300),
    col3 VARCHAR(300),
    col4 VARCHAR(300),
    col5 VARCHAR(300),
    col6 VARCHAR(300),
    col7 VARCHAR(300),
    col8 VARCHAR(300),
    col9 VARCHAR(300),
    col10 VARCHAR(300)
) ROW_FORMAT=REDUNDANT;

-- ERROR with strict mode:
-- Row size too large (> 8126). Changing ROW_FORMAT to DYNAMIC or COMPRESSED
```

The fix is to use `ROW_FORMAT=DYNAMIC` or `ROW_FORMAT=COMPRESSED`, which store long columns off-page:

```sql
CREATE TABLE wide_table (
    col1 VARCHAR(300),
    -- ... other columns
) ROW_FORMAT=DYNAMIC;
```

## Correct ROW_FORMAT and KEY_BLOCK_SIZE Combinations

```sql
-- Valid: COMPRESSED with KEY_BLOCK_SIZE
CREATE TABLE compressed_table (
    id INT PRIMARY KEY,
    data TEXT
) ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8;

-- Valid: DYNAMIC without KEY_BLOCK_SIZE
CREATE TABLE dynamic_table (
    id INT PRIMARY KEY,
    data TEXT
) ROW_FORMAT=DYNAMIC;
```

## Checking Table Formats

```sql
-- Review table format settings
SELECT TABLE_NAME, ROW_FORMAT, CREATE_OPTIONS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb'
ORDER BY TABLE_NAME;
```

## Summary

`innodb_strict_mode=ON` (the default since MySQL 5.7.7) converts certain DDL warnings into errors, catching misconfigured table definitions at creation time rather than at runtime. It validates `KEY_BLOCK_SIZE` and `ROW_FORMAT` compatibility and enforces row size limits. Keep strict mode enabled in all environments to ensure your DDL is valid before deploying to production.
