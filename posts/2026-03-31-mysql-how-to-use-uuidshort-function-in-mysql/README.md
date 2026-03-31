# How to Use UUID_SHORT() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, UUID_SHORT, Primary Key, SQL Functions

Description: Learn how to use the UUID_SHORT() function in MySQL to generate unique 64-bit integer identifiers suitable for use as compact primary keys.

---

## What is UUID_SHORT()

MySQL's `UUID_SHORT()` function generates a 64-bit unsigned integer that is unique within a single MySQL server and guaranteed to be monotonically increasing within a session. It is more compact and indexable than `UUID()` while still being unique.

Syntax:

```sql
UUID_SHORT()
```

Return type: `BIGINT UNSIGNED`

## How UUID_SHORT() Works

The value is composed of:
- `server_id` (bits 56-63): 8 bits from the server's `server_id` variable
- `seconds` since epoch (bits 24-55): 32 bits
- incrementing counter (bits 0-23): 24 bits

This means `UUID_SHORT()` generates values that are roughly time-ordered and server-specific.

## Basic Usage

```sql
SELECT UUID_SHORT();
-- Output: 92395783831158784 (example 64-bit integer)

-- Each call returns a different, larger value
SELECT UUID_SHORT(), UUID_SHORT(), UUID_SHORT();
```

## Using UUID_SHORT() as a Primary Key

```sql
CREATE TABLE sessions (
  id BIGINT UNSIGNED NOT NULL DEFAULT (UUID_SHORT()),
  user_id INT NOT NULL,
  created_at DATETIME DEFAULT NOW(),
  PRIMARY KEY (id)
);

INSERT INTO sessions (user_id) VALUES (42);
SELECT LAST_INSERT_ID();  -- Works normally with integer PKs
```

## UUID_SHORT() vs UUID() vs AUTO_INCREMENT

```sql
SELECT UUID_SHORT() AS short_uuid;  -- 64-bit integer
SELECT UUID() AS full_uuid;          -- 36-char string
SELECT 1 AS auto_increment_example;  -- Integer, sequential per table

-- UUID_SHORT output example: 92395783831158784
-- UUID output example:       550e8400-e29b-41d4-a716-446655440000
```

| Feature | UUID_SHORT() | UUID() | AUTO_INCREMENT |
|---|---|---|---|
| Type | BIGINT (8 bytes) | CHAR(36) / BINARY(16) | INT/BIGINT |
| Globally unique | Per-server | Globally | Per-table |
| Sequential | Roughly | No | Yes |
| Index performance | Good | Poor (string) / OK (binary) | Best |

## Verifying UUID_SHORT() Uniqueness

The uniqueness depends on `server_id` being unique across servers:

```sql
-- Check the current server_id
SHOW VARIABLES LIKE 'server_id';
```

On different servers, set unique server IDs:

```text
[mysqld]
server_id = 1   -- Server 1
server_id = 2   -- Server 2
```

Note: If two servers share the same `server_id` and generate IDs in the same second, collisions are possible.

## Limitations

- Not globally unique across servers with the same `server_id`
- `UUID_SHORT()` generates approximately 16 million unique values per second per server (24-bit counter)
- Not suitable for distributed systems without ensuring unique `server_id` values

## Comparing UUID_SHORT() to BIGINT AUTO_INCREMENT

Both produce integer primary keys. The difference:

```sql
-- AUTO_INCREMENT: sequential per table, simple
CREATE TABLE t1 (id BIGINT AUTO_INCREMENT PRIMARY KEY, ...);

-- UUID_SHORT: time-based, server-aware
CREATE TABLE t2 (id BIGINT UNSIGNED DEFAULT (UUID_SHORT()), ...);
```

Use `UUID_SHORT()` when you need IDs that work across multiple MySQL instances (multi-master) or when merging data from separate servers.

## Practical Use Case - Distributed Session IDs

```sql
CREATE TABLE user_sessions (
  session_id BIGINT UNSIGNED NOT NULL,
  user_id INT NOT NULL,
  ip_address VARCHAR(45),
  created_at DATETIME DEFAULT NOW(),
  PRIMARY KEY (session_id)
);

-- Application generates session ID using UUID_SHORT
INSERT INTO user_sessions (session_id, user_id, ip_address)
VALUES (UUID_SHORT(), 101, '192.168.1.10');
```

## Summary

`UUID_SHORT()` generates a compact 64-bit integer that is unique within a MySQL server instance, making it a good middle ground between `AUTO_INCREMENT` and full `UUID()`. It produces time-ordered values that are index-friendly, avoiding the B-tree fragmentation caused by random UUID strings. Use it for applications that need compact, unique IDs across multiple tables or when merging data from multiple servers with distinct `server_id` values.
