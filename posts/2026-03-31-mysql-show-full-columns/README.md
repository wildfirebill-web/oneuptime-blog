# How to Use SHOW FULL COLUMNS in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Column, Schema

Description: Learn how to use SHOW FULL COLUMNS in MySQL to view detailed column metadata including data types, collation, privileges, and comments.

---

## What Is SHOW FULL COLUMNS

`SHOW FULL COLUMNS` (a variant of `SHOW COLUMNS`) returns detailed metadata about each column in a table. Unlike the basic `SHOW COLUMNS` or `DESCRIBE` statements, the `FULL` modifier adds the `Collation`, `Privileges`, and `Comment` columns to the output, giving a more complete picture of the table's schema.

```sql
SHOW FULL COLUMNS FROM table_name;
SHOW FULL COLUMNS FROM table_name FROM database_name;
SHOW FULL COLUMNS FROM table_name LIKE 'column_pattern';
SHOW FULL COLUMNS FROM table_name WHERE condition;
```

## Basic Usage

```sql
USE myapp_db;
SHOW FULL COLUMNS FROM users;
```

```text
+-----------+--------------+--------------------+------+-----+---------+----------------+---------------------------------+---------+
| Field     | Type         | Collation          | Null | Key | Default | Extra          | Privileges                      | Comment |
+-----------+--------------+--------------------+------+-----+---------+----------------+---------------------------------+---------+
| id        | int          | NULL               | NO   | PRI | NULL    | auto_increment | select,insert,update,references |         |
| username  | varchar(100) | utf8mb4_0900_ai_ci | NO   | UNI | NULL    |                | select,insert,update,references |         |
| email     | varchar(255) | utf8mb4_0900_ai_ci | NO   | UNI | NULL    |                | select,insert,update,references |         |
| created_at| datetime     | NULL               | NO   |     | NULL    |                | select,insert,update,references |         |
+-----------+--------------+--------------------+------+-----+---------+----------------+---------------------------------+---------+
```

## Output Columns Explained

- **Field**: Column name
- **Type**: Data type (e.g., `int`, `varchar(255)`, `datetime`)
- **Collation**: Character set collation for string columns; `NULL` for non-string types
- **Null**: Whether `NULL` values are allowed (`YES` or `NO`)
- **Key**: Index type (`PRI` for primary key, `UNI` for unique, `MUL` for non-unique index)
- **Default**: Default value for the column
- **Extra**: Additional info like `auto_increment`, `DEFAULT_GENERATED`, `on update CURRENT_TIMESTAMP`
- **Privileges**: Column-level privileges for the current user
- **Comment**: Column comment set with `COMMENT` clause

## Filtering with LIKE and WHERE

Find columns matching a name pattern:

```sql
-- Find columns with 'email' in their name
SHOW FULL COLUMNS FROM users LIKE '%email%';

-- Find nullable columns
SHOW FULL COLUMNS FROM orders WHERE `Null` = 'YES';

-- Find columns with character collation
SHOW FULL COLUMNS FROM articles WHERE Collation IS NOT NULL;
```

## Checking Column Comments

Column comments are invaluable for documentation. To find columns with comments:

```sql
SHOW FULL COLUMNS FROM orders WHERE Comment != '';
```

To add comments to columns:

```sql
ALTER TABLE orders
  MODIFY COLUMN status VARCHAR(20) NOT NULL COMMENT 'Order lifecycle status: pending, processing, shipped, delivered, cancelled';
```

## Using information_schema for Programmatic Access

For scripting or reporting, query `information_schema.COLUMNS`:

```sql
SELECT
  COLUMN_NAME,
  DATA_TYPE,
  CHARACTER_MAXIMUM_LENGTH,
  IS_NULLABLE,
  COLUMN_DEFAULT,
  EXTRA,
  COLUMN_COMMENT,
  COLLATION_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'myapp_db'
  AND TABLE_NAME = 'users'
ORDER BY ORDINAL_POSITION;
```

## Auditing Collation Consistency

Use `SHOW FULL COLUMNS` to find columns with inconsistent collations (which can cause silent comparison issues):

```sql
SELECT TABLE_NAME, COLUMN_NAME, COLLATION_NAME
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = 'myapp_db'
  AND COLLATION_NAME IS NOT NULL
  AND COLLATION_NAME != 'utf8mb4_0900_ai_ci'
ORDER BY TABLE_NAME, COLUMN_NAME;
```

## Comparing SHOW COLUMNS vs SHOW FULL COLUMNS

```sql
-- Basic output - no Collation, Privileges, or Comment
SHOW COLUMNS FROM users;

-- Full output - includes all metadata
SHOW FULL COLUMNS FROM users;

-- Equivalent to SHOW COLUMNS
DESCRIBE users;
```

## Summary

`SHOW FULL COLUMNS` provides comprehensive column metadata beyond what `DESCRIBE` or basic `SHOW COLUMNS` offers - including collation, per-column privileges, and developer comments. Use it to audit schema documentation, verify character set consistency, and inspect column-level access controls. For automation and cross-table analysis, `information_schema.COLUMNS` provides the same data in a queryable format.
