# How to Use INSERT IGNORE in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Insert Ignore, Duplicate Key, Sql, Database

Description: Learn how to use INSERT IGNORE in MySQL to silently skip rows that would cause duplicate key or constraint errors during insert operations.

---

## Introduction

`INSERT IGNORE` in MySQL instructs the server to silently ignore errors that would normally prevent an insert - most commonly duplicate key violations on PRIMARY KEY or UNIQUE constraints. Instead of raising an error, MySQL skips the conflicting row and continues inserting the remaining rows.

## Basic Syntax

```sql
INSERT IGNORE INTO table_name (column1, column2, ...)
VALUES (value1, value2, ...);
```

## Simple INSERT IGNORE Example

```sql
-- users table has UNIQUE constraint on email
INSERT IGNORE INTO users (id, name, email)
VALUES (1, 'Alice', 'alice@example.com');

-- If alice@example.com already exists, this is silently skipped
INSERT IGNORE INTO users (id, name, email)
VALUES (2, 'Bob', 'alice@example.com');
```

Without `IGNORE`, the second insert would raise an error. With `IGNORE`, it is silently discarded.

## Bulk INSERT IGNORE

`INSERT IGNORE` is especially useful for bulk inserts where some rows may already exist:

```sql
INSERT IGNORE INTO tags (name)
VALUES
  ('mysql'),
  ('sql'),
  ('database'),
  ('mysql');  -- duplicate, will be skipped
```

## INSERT IGNORE vs ON DUPLICATE KEY UPDATE

- `INSERT IGNORE`: skips the row entirely if a conflict exists. Original data unchanged.
- `ON DUPLICATE KEY UPDATE`: updates specified columns of the existing row.

```sql
-- INSERT IGNORE: existing row is untouched
INSERT IGNORE INTO products (sku, price) VALUES ('ABC', 19.99);

-- ON DUPLICATE KEY UPDATE: updates existing row
INSERT INTO products (sku, price) VALUES ('ABC', 19.99)
ON DUPLICATE KEY UPDATE price = 19.99;
```

Use `INSERT IGNORE` when you want to preserve the existing row's data unchanged.

## INSERT IGNORE for Bulk Data Loading

When importing large datasets where some records may already exist:

```sql
LOAD DATA INFILE '/data/products.csv'
IGNORE INTO TABLE products
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
(sku, name, price, category);
```

Or with SELECT:

```sql
INSERT IGNORE INTO products_new
SELECT * FROM products_staging;
```

## Errors Ignored by INSERT IGNORE

`INSERT IGNORE` suppresses several error types:
- Duplicate PRIMARY KEY errors.
- Duplicate UNIQUE KEY errors.
- Data truncation errors (converts to warnings).
- Row too large errors.

```sql
-- Data truncation becomes a warning, not an error
INSERT IGNORE INTO names (short_name)
VALUES ('This is a very long string that exceeds the column limit');
-- Truncated and inserted, not rejected
```

## Checking Warnings After INSERT IGNORE

After an INSERT IGNORE, check for warnings to understand what was skipped:

```sql
INSERT IGNORE INTO users (id, email) VALUES (1, 'test@example.com');
SHOW WARNINGS;
```

Output:

```text
+-------+------+-----------------------------------+
| Level | Code | Message                           |
+-------+------+-----------------------------------+
| Note  | 1062 | Duplicate entry '1' for key 'PRIMARY' |
+-------+------+-----------------------------------+
```

## Checking Affected Rows

The number of rows actually inserted vs skipped:

```sql
INSERT IGNORE INTO tags (name) VALUES ('a'), ('b'), ('a');
SELECT ROW_COUNT(); -- returns 2 (one duplicate skipped)
```

## When NOT to Use INSERT IGNORE

Avoid `INSERT IGNORE` when:
- You need to know which rows were skipped.
- Silent data truncation is unacceptable.
- You need to update existing rows (use `ON DUPLICATE KEY UPDATE` instead).
- You want strict error handling for data integrity.

## INSERT IGNORE with Multiple Unique Keys

If a table has multiple UNIQUE constraints, a row is skipped if it violates ANY of them.

```sql
CREATE TABLE sessions (
  session_id INT PRIMARY KEY,
  token VARCHAR(255) UNIQUE,
  user_id INT,
  UNIQUE KEY unique_user_session (user_id, token)
);

INSERT IGNORE INTO sessions (session_id, token, user_id)
VALUES (1, 'abc123', 42);
```

## Summary

`INSERT IGNORE` is a simple way to skip conflicting rows during inserts without raising errors. It is ideal for bulk imports, data syncing, and populating lookup tables where duplicates may exist. Use it when you want to preserve existing rows unchanged, and check `SHOW WARNINGS` afterward to audit what was skipped.
