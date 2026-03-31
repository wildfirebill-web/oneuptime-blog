# How to Use BIT_COUNT() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Bitwise, Function, Integer

Description: Learn how to use the BIT_COUNT() function in MySQL to count the number of set bits in an integer, useful for permissions, analytics, and data validation.

---

## What Is BIT_COUNT()?

`BIT_COUNT()` is a MySQL function that counts how many bits are set to 1 in the binary representation of an integer. It is also known as the "popcount" or "Hamming weight" function and is useful for analyzing bitmask columns, validating flag combinations, and summarizing compact integer data.

## Basic Syntax

```sql
BIT_COUNT(N)
```

`N` is treated as a 64-bit unsigned integer.

## Simple Examples

```sql
-- 5 = 101 in binary (two 1s)
SELECT BIT_COUNT(5);    -- Result: 2

-- 255 = 11111111 (eight 1s)
SELECT BIT_COUNT(255);  -- Result: 8

-- 0 has no set bits
SELECT BIT_COUNT(0);    -- Result: 0

-- 1 has one set bit
SELECT BIT_COUNT(1);    -- Result: 1

-- ~0 is all 64 bits set
SELECT BIT_COUNT(~0);   -- Result: 64
```

## Counting Permissions

Using the bitmask permission pattern:

```sql
-- READ=1, WRITE=2, DELETE=4, ADMIN=8

CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  permissions INT UNSIGNED DEFAULT 0
);

INSERT INTO users (name, permissions) VALUES
  ('alice', 11),  -- READ + WRITE + ADMIN = 3 permissions
  ('bob',   1),   -- READ only = 1 permission
  ('carol', 7),   -- READ + WRITE + DELETE = 3 permissions
  ('dave',  15);  -- all 4 permissions

-- Count how many permissions each user has
SELECT name, permissions, BIT_COUNT(permissions) AS permission_count
FROM users
ORDER BY permission_count DESC;
```

## Filtering by Number of Set Bits

```sql
-- Find users with exactly 2 permissions set
SELECT name, permissions
FROM users
WHERE BIT_COUNT(permissions) = 2;

-- Find users with 3 or more permissions
SELECT name, permissions
FROM users
WHERE BIT_COUNT(permissions) >= 3;
```

## Aggregating Bit Counts

```sql
-- Average number of flags set per user
SELECT AVG(BIT_COUNT(flags)) AS avg_flags_set
FROM user_settings;

-- Distribution of permission counts
SELECT BIT_COUNT(permissions) AS num_permissions, COUNT(*) AS user_count
FROM users
GROUP BY BIT_COUNT(permissions)
ORDER BY num_permissions;
```

## Validating Bitmask Columns

```sql
-- Flag rows where more than 4 bits are set (may indicate data corruption)
SELECT id, value, BIT_COUNT(value) AS set_bits
FROM packed_data
WHERE BIT_COUNT(value) > 4;
```

## BIT_COUNT with XOR to Compare Integers

XOR two values and count differing bits (Hamming distance):

```sql
-- How many bits differ between two permission sets?
SELECT BIT_COUNT(permissions_old ^ permissions_new) AS bits_changed
FROM permission_change_log;
```

## NULL Handling

```sql
-- BIT_COUNT(NULL) returns NULL
SELECT BIT_COUNT(NULL);  -- Result: NULL

-- Handle with COALESCE
SELECT BIT_COUNT(COALESCE(permissions, 0)) FROM users;
```

## Summary

`BIT_COUNT()` is a concise MySQL function for counting the number of 1-bits in an integer. It is most valuable when working with bitmask columns - letting you filter users by how many flags they have set, compute averages, or validate that packed integer fields contain the expected number of active bits. Combine it with `^` (XOR) to compute the Hamming distance between two integer values.
