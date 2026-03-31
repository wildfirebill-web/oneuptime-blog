# How to Use UUID_TO_BIN() and BIN_TO_UUID() Functions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Function, UUID, Performance, Binary

Description: Learn how to use UUID_TO_BIN() and BIN_TO_UUID() in MySQL to store UUIDs as compact BINARY(16) values for better index performance.

---

## Overview

MySQL's `UUID_TO_BIN()` and `BIN_TO_UUID()` functions, introduced in MySQL 8.0, allow you to convert UUID strings to compact binary representations and back. Storing UUIDs as `BINARY(16)` rather than `CHAR(36)` saves space and dramatically improves index performance.

## Why Store UUIDs as Binary

A UUID in string form occupies 36 bytes (including hyphens). The same UUID stored as `BINARY(16)` occupies only 16 bytes. For large tables with UUID primary keys, this difference in index size has a significant impact on query speed and memory usage.

## UUID_TO_BIN() Function

`UUID_TO_BIN(uuid)` converts a UUID string to a 16-byte binary value. An optional second argument controls whether the time-related components are rearranged for better sequential insertion performance:

```sql
-- Basic conversion
SELECT UUID_TO_BIN('6ccd780c-baba-1026-9564-5b8c656024db');

-- With swap_flag = 1 for sequential-friendly ordering
SELECT UUID_TO_BIN('6ccd780c-baba-1026-9564-5b8c656024db', 1);
```

When `swap_flag = 1`, the time_low and time_high fields of version-1 UUIDs are swapped, making the values more sequential and friendlier for clustered indexes in InnoDB.

## BIN_TO_UUID() Function

`BIN_TO_UUID(binary_uuid)` is the inverse operation - it converts a `BINARY(16)` value back to a human-readable UUID string:

```sql
-- Convert binary back to string
SELECT BIN_TO_UUID(uuid_col) FROM users LIMIT 5;

-- With swap_flag to reverse swapping
SELECT BIN_TO_UUID(uuid_col, 1) FROM users LIMIT 5;
```

## Creating a Table with Binary UUID Primary Key

```sql
CREATE TABLE users (
  id    BINARY(16)   NOT NULL DEFAULT (UUID_TO_BIN(UUID(), 1)),
  name  VARCHAR(100) NOT NULL,
  email VARCHAR(255) NOT NULL,
  PRIMARY KEY (id)
);
```

## Inserting and Querying Data

```sql
-- Insert with explicit UUID
INSERT INTO users (id, name, email)
VALUES (UUID_TO_BIN(UUID(), 1), 'Alice', 'alice@example.com');

-- Query by UUID string
SELECT BIN_TO_UUID(id, 1) AS uuid, name, email
FROM users
WHERE id = UUID_TO_BIN('6ccd780c-baba-1026-9564-5b8c656024db', 1);
```

## Generating and Displaying UUIDs

For application code, it is common to generate the UUID at the application layer and pass it in:

```sql
SET @new_uuid = UUID();

INSERT INTO orders (id, customer_id, total)
VALUES (UUID_TO_BIN(@new_uuid, 1), 42, 99.99);

-- Retrieve by the same UUID
SELECT BIN_TO_UUID(id, 1) AS order_id, total
FROM orders
WHERE id = UUID_TO_BIN(@new_uuid, 1);
```

## Performance Comparison

```sql
-- Comparing storage sizes
SELECT
  LENGTH(UUID())                        AS uuid_string_bytes,  -- 36
  LENGTH(UUID_TO_BIN(UUID()))           AS uuid_binary_bytes;  -- 16
```

Using `BINARY(16)` reduces index memory usage by more than 55% compared to `CHAR(36)`, which matters greatly at millions of rows.

## Summary

`UUID_TO_BIN()` and `BIN_TO_UUID()` make it practical to use UUIDs as primary keys in high-performance MySQL tables. By converting UUIDs to 16-byte binary values - and optionally reordering time fields for sequential inserts - you get the global uniqueness of UUIDs without sacrificing index efficiency.
