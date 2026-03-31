# How to Use SHOW COLUMNS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, DDL, Column, Schema, Information Schema

Description: Learn how to use SHOW COLUMNS and SHOW FULL COLUMNS in MySQL to inspect column definitions, collations, privileges, and comments for any table or view.

---

## Basic Syntax

`SHOW COLUMNS` returns metadata about every column in a table, including the data type, nullability, key participation, default value, and extra attributes.

```sql
SHOW COLUMNS FROM orders;
SHOW COLUMNS FROM orders FROM your_database;  -- specify database explicitly
```

```text
+-------------+--------------+------+-----+-------------------+-------------------+
| Field       | Type         | Null | Key | Default           | Extra             |
+-------------+--------------+------+-----+-------------------+-------------------+
| id          | int          | NO   | PRI | NULL              | auto_increment    |
| customer_id | int          | NO   | MUL | NULL              |                   |
| status      | varchar(20)  | NO   |     | pending           |                   |
| total       | decimal(12,2)| NO   |     | NULL              |                   |
| created_at  | datetime     | NO   |     | CURRENT_TIMESTAMP | DEFAULT_GENERATED |
+-------------+--------------+------+-----+-------------------+-------------------+
```

## SHOW FULL COLUMNS

The `FULL` modifier adds three extra columns: `Collation`, `Privileges`, and `Comment`.

```sql
SHOW FULL COLUMNS FROM orders;
```

```text
+-------------+-------------+-----------+------+...+-------------------+-----------+---------+
| Field       | Type        | Collation | Null |...| Extra             | Privileges| Comment |
+-------------+-------------+-----------+------+...+-------------------+-----------+---------+
| id          | int         | NULL      | NO   |...| auto_increment    | ...       |         |
| status      | varchar(20) | utf8mb4.. | NO   |...| DEFAULT_GENERATED | ...       | Order state |
+-------------+-------------+-----------+------+...+-------------------+-----------+---------+
```

## Filtering with LIKE or WHERE

```sql
-- Only columns whose names start with 'order'
SHOW COLUMNS FROM shipments LIKE 'order%';

-- Columns that allow NULL
SHOW COLUMNS FROM orders WHERE `Null` = 'YES';
```

## Relationship to DESCRIBE

`SHOW COLUMNS` and `DESCRIBE` (or `DESC`) are equivalent for basic output. `SHOW FULL COLUMNS` is the superset - it reveals collation, privileges, and comments that `DESCRIBE` omits.

## Querying information_schema for More Flexibility

For scripting or joining with other metadata:

```sql
SELECT
    COLUMN_NAME,
    COLUMN_TYPE,
    CHARACTER_SET_NAME,
    COLLATION_NAME,
    IS_NULLABLE,
    COLUMN_KEY,
    COLUMN_DEFAULT,
    EXTRA,
    COLUMN_COMMENT
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'your_database'
  AND TABLE_NAME = 'orders'
ORDER BY ORDINAL_POSITION;
```

## Listing Columns Across All Tables

```sql
SELECT TABLE_NAME, COLUMN_NAME, COLUMN_TYPE
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'your_database'
ORDER BY TABLE_NAME, ORDINAL_POSITION;
```

## Finding Columns with No Default Value

```sql
SELECT TABLE_NAME, COLUMN_NAME, COLUMN_TYPE
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'your_database'
  AND IS_NULLABLE = 'NO'
  AND COLUMN_DEFAULT IS NULL
  AND EXTRA NOT LIKE '%auto_increment%';
```

This query helps catch columns that could cause `INSERT` failures if not explicitly provided.

## Using from the Command Line

```bash
mysql -u root -p your_database -e "SHOW FULL COLUMNS FROM orders\G"
```

## Summary

`SHOW COLUMNS` and its `FULL` variant are the fastest ways to inspect column definitions in MySQL. Use `SHOW FULL COLUMNS` when you need collation and comment details. For more powerful filtering, sorting, or joining across tables, query `information_schema.COLUMNS` directly.
