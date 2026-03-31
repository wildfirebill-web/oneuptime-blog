# How to Query INFORMATION_SCHEMA.COLUMNS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, INFORMATION_SCHEMA, Column, Schema, Metadata

Description: Learn how to query INFORMATION_SCHEMA.COLUMNS in MySQL to inspect column definitions, data types, nullability, defaults, and auto_increment columns.

---

## Overview

`INFORMATION_SCHEMA.COLUMNS` contains metadata about every column in every table across your MySQL instance. It is essential for schema documentation, data type audits, migration planning, and automated code generation.

## Basic Column Listing

```sql
SELECT
  COLUMN_NAME,
  DATA_TYPE,
  COLUMN_TYPE,
  IS_NULLABLE,
  COLUMN_DEFAULT,
  COLUMN_KEY,
  EXTRA
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'myapp'
  AND TABLE_NAME = 'orders'
ORDER BY ORDINAL_POSITION;
```

## Key Columns

| Column | Description |
|--------|-------------|
| `TABLE_SCHEMA` | Database name |
| `TABLE_NAME` | Table name |
| `COLUMN_NAME` | Column name |
| `ORDINAL_POSITION` | Column position (1-based) |
| `COLUMN_DEFAULT` | Default value |
| `IS_NULLABLE` | YES or NO |
| `DATA_TYPE` | Base type (varchar, int, etc.) |
| `COLUMN_TYPE` | Full type including length/precision |
| `CHARACTER_MAXIMUM_LENGTH` | Max length for string types |
| `NUMERIC_PRECISION` | Precision for numeric types |
| `COLUMN_KEY` | PRI, UNI, MUL, or empty |
| `EXTRA` | auto_increment, DEFAULT_GENERATED, etc. |
| `COLUMN_COMMENT` | Column comment |

## Finding Nullable Columns in Primary Key Tables

```sql
SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME, DATA_TYPE
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'myapp'
  AND IS_NULLABLE = 'YES'
  AND COLUMN_KEY = 'PRI';
```

## Finding All VARCHAR Columns with Their Max Lengths

```sql
SELECT
  TABLE_NAME,
  COLUMN_NAME,
  CHARACTER_MAXIMUM_LENGTH
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'myapp'
  AND DATA_TYPE = 'varchar'
ORDER BY TABLE_NAME, ORDINAL_POSITION;
```

## Auditing Text Columns That Should Use utf8mb4

```sql
SELECT
  TABLE_SCHEMA,
  TABLE_NAME,
  COLUMN_NAME,
  CHARACTER_SET_NAME,
  COLLATION_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'myapp'
  AND DATA_TYPE IN ('varchar', 'text', 'char')
  AND CHARACTER_SET_NAME != 'utf8mb4';
```

## Finding All AUTO_INCREMENT Columns

```sql
SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME, COLUMN_TYPE
FROM information_schema.COLUMNS
WHERE EXTRA LIKE '%auto_increment%';
```

## Generating Column Definitions for Documentation

```sql
SELECT
  CONCAT(
    COLUMN_NAME, ' ',
    COLUMN_TYPE,
    IF(IS_NULLABLE = 'NO', ' NOT NULL', ' NULL'),
    IF(COLUMN_DEFAULT IS NOT NULL, CONCAT(' DEFAULT ', QUOTE(COLUMN_DEFAULT)), ''),
    IF(EXTRA != '', CONCAT(' ', EXTRA), ''),
    IF(COLUMN_COMMENT != '', CONCAT(' COMMENT ', QUOTE(COLUMN_COMMENT)), '')
  ) AS column_definition
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'myapp'
  AND TABLE_NAME = 'orders'
ORDER BY ORDINAL_POSITION;
```

## Finding Tables Missing a created_at Column

```sql
SELECT TABLE_NAME
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'myapp'
  AND TABLE_TYPE = 'BASE TABLE'
  AND TABLE_NAME NOT IN (
    SELECT TABLE_NAME
    FROM information_schema.COLUMNS
    WHERE TABLE_SCHEMA = 'myapp'
      AND COLUMN_NAME = 'created_at'
  );
```

## Summary

`INFORMATION_SCHEMA.COLUMNS` is the authoritative source for column-level metadata in MySQL. By querying it, you can audit data types, validate nullability constraints, identify charset inconsistencies, and generate schema documentation automatically. It is especially useful during migrations and schema review processes.
