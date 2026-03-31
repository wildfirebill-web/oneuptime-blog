# How to Show Table Structure with DESCRIBE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, DDL, Table, Schema, Information Schema

Description: Learn how to use DESCRIBE and DESC in MySQL to inspect table structure, column types, nullability, defaults, keys, and extra attributes quickly.

---

## The DESCRIBE Command

`DESCRIBE` (or its shorthand `DESC`) displays the structure of a MySQL table. It shows each column's name, data type, nullability, key participation, default value, and any extra attributes such as `auto_increment`.

```sql
DESCRIBE orders;
-- or equivalently:
DESC orders;
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

## Understanding the Output Columns

| Column | Meaning |
|--------|---------|
| Field | Column name |
| Type | Data type and length |
| Null | YES if the column accepts NULL |
| Key | PRI (primary), UNI (unique), MUL (non-unique index) |
| Default | Default value, or NULL if none |
| Extra | auto_increment, on update CURRENT_TIMESTAMP, etc. |

## Using DESCRIBE with a Column Filter

You can narrow the output to a specific column or pattern:

```sql
DESCRIBE orders 'customer%';
```

This shows only columns whose names match the `customer%` pattern.

## Equivalent: SHOW COLUMNS

`SHOW COLUMNS` is functionally identical to `DESCRIBE`:

```sql
SHOW COLUMNS FROM orders;
SHOW FULL COLUMNS FROM orders;  -- also shows Collation, Privileges, and Comment
```

## Equivalent: information_schema Query

For programmatic access or more filtering options:

```sql
SELECT
    COLUMN_NAME,
    COLUMN_TYPE,
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

## Describing a View

`DESCRIBE` works on views too:

```sql
CREATE VIEW active_orders AS
    SELECT id, customer_id, total FROM orders WHERE status = 'pending';

DESCRIBE active_orders;
```

## Describing a Stored Procedure or Function

For routines, use `SHOW CREATE PROCEDURE` or `SHOW CREATE FUNCTION` instead - `DESCRIBE` does not apply to them.

```sql
SHOW CREATE PROCEDURE update_order_status\G
```

## Practical Usage in Scripts

```bash
mysql -u root -p your_database -e "DESCRIBE orders"
```

This is useful for quick schema validation in deployment scripts or CI pipelines.

## Summary

`DESCRIBE` and `DESC` give a fast, human-readable overview of any MySQL table or view structure. Use `SHOW FULL COLUMNS` when you need collation and comment details. For automation and more complex filtering, query `information_schema.COLUMNS` directly.
