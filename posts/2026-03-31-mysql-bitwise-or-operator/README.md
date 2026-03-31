# How to Use Bitwise OR (|) Operator in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Bitwise, Operator, Integer

Description: Learn how to use the Bitwise OR (|) operator in MySQL to combine bit flags, set specific bits, and merge permission values stored as integers.

---

## What Is the Bitwise OR Operator?

The bitwise OR operator (`|`) in MySQL compares each bit of two integer values and returns a 1 in each position where at least one operand has a 1. It is the complement to bitwise AND and is most commonly used to set bits or combine flag values.

## Basic Syntax

```sql
expr1 | expr2
```

Both operands are treated as 64-bit unsigned integers.

## Simple Examples

```sql
-- Basic OR operation
SELECT 12 | 10;
-- 12 = 1100
-- 10 = 1010
-- |  = 1110 = 14
-- Result: 14

SELECT 5 | 3;
-- 5 = 101
-- 3 = 011
-- | = 111 = 7
-- Result: 7
```

## Practical Use Case: Setting Permission Flags

Using the same bitmask pattern as with `&`:

```sql
-- Permissions: READ=1, WRITE=2, DELETE=4, ADMIN=8

CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  permissions INT UNSIGNED DEFAULT 0
);

-- Start with no permissions
INSERT INTO users (name, permissions) VALUES ('carol', 0);
```

## Adding a Permission with OR

```sql
-- Grant READ (1) to carol
UPDATE users
SET permissions = permissions | 1
WHERE name = 'carol';

-- Grant WRITE (2) as well
UPDATE users
SET permissions = permissions | 2
WHERE name = 'carol';

-- Verify: should now be 3 (READ + WRITE)
SELECT name, permissions FROM users WHERE name = 'carol';
```

## Combining Multiple Flags at Once

```sql
-- Grant READ + ADMIN in one step (1 | 8 = 9)
UPDATE users
SET permissions = permissions | 9
WHERE name = 'bob';

-- Assign READ + WRITE + DELETE in INSERT
INSERT INTO users (name, permissions)
VALUES ('dave', 1 | 2 | 4);  -- = 7
```

## Using Bitwise OR in SELECT

```sql
-- Combine two flag columns into one
SELECT
  id,
  name,
  (system_flags | user_flags) AS combined_flags
FROM user_settings;
```

## Idempotent Flag Setting

OR is idempotent for bit setting - applying it twice has no additional effect:

```sql
-- Setting the same bit twice is safe
UPDATE users SET permissions = permissions | 2 WHERE name = 'carol';
UPDATE users SET permissions = permissions | 2 WHERE name = 'carol';
-- permissions is still 3, not 5
```

## Working with Multiple Values

```sql
-- Find all users that have any of the flags 1, 2, or 4 set
SELECT name, permissions
FROM users
WHERE (permissions & (1 | 2 | 4)) > 0;
```

## NULL Handling

```sql
-- OR with NULL returns NULL
SELECT NULL | 5;   -- Result: NULL

-- Use COALESCE to handle nullable columns
SELECT COALESCE(permissions, 0) | 2 FROM users;
```

## Summary

The Bitwise OR (`|`) operator is the primary tool in MySQL for setting bits and combining flag values. When building permission systems or feature toggles stored as integers, use `column | flag` to add a permission without affecting others. It is safe to apply the same OR operation multiple times since setting an already-set bit has no effect, making it a reliable pattern for idempotent updates.
