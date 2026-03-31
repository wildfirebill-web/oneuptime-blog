# How to Use MAKE_SET() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, String Function, Database

Description: Learn how MySQL's MAKE_SET() function builds a comma-separated set string from a bitmask integer and a list of string arguments.

---

## What is MAKE_SET()?

`MAKE_SET()` returns a comma-separated string containing the strings from the argument list that correspond to set bits in a bitmask integer. It is the complement of the `SET` data type's bit manipulation behavior and is useful for converting bitmask-encoded flags into readable string representations.

The syntax is:

```sql
MAKE_SET(bits, str1, str2, ..., strN)
```

`bits` is an integer where each bit position (starting from bit 0 for str1) controls whether that string is included in the output.

## How Bitmask Mapping Works

| Bit value | Meaning                 |
|-----------|-------------------------|
| 1 (bit 0) | Include str1            |
| 2 (bit 1) | Include str2            |
| 4 (bit 2) | Include str3            |
| 8 (bit 3) | Include str4            |

Bits combine: a value of 3 (binary `011`) includes str1 and str2.

## Basic Examples

```sql
SELECT MAKE_SET(1, 'a', 'b', 'c');
-- Result: a

SELECT MAKE_SET(3, 'a', 'b', 'c');
-- Result: a,b

SELECT MAKE_SET(5, 'a', 'b', 'c');
-- Result: a,c

SELECT MAKE_SET(7, 'a', 'b', 'c');
-- Result: a,b,c
```

## NULL Handling

NULL strings in the argument list are silently skipped - they do not appear in the output even if their corresponding bit is set:

```sql
SELECT MAKE_SET(3, 'read', NULL, 'write');
-- Result: read
```

If `bits` itself is `NULL`, the result is `NULL`.

## Practical Use Case - User Permission Flags

Suppose user permissions are stored as a bitmask integer:

```sql
CREATE TABLE users (
  id INT PRIMARY KEY,
  username VARCHAR(50),
  permissions INT  -- bitmask: 1=read, 2=write, 4=delete, 8=admin
);

INSERT INTO users VALUES (1, 'alice', 7), (2, 'bob', 1), (3, 'carol', 11);

SELECT
  username,
  permissions,
  MAKE_SET(permissions, 'read', 'write', 'delete', 'admin') AS permission_labels
FROM users;
```

```text
alice | 7  | read,write,delete
bob   | 1  | read
carol | 11 | read,write,admin
```

## Using MAKE_SET() with Feature Flags

```sql
SELECT
  product_name,
  MAKE_SET(feature_flags, 'warranty', 'express_shipping', 'gift_wrap', 'installation') AS features
FROM products
WHERE category = 'Electronics';
```

## Combining MAKE_SET() with FIND_IN_SET()

`MAKE_SET()` and `FIND_IN_SET()` are natural partners. Build the set string with `MAKE_SET()` and then search it with `FIND_IN_SET()`:

```sql
SELECT *
FROM users
WHERE FIND_IN_SET('admin', MAKE_SET(permissions, 'read', 'write', 'delete', 'admin')) > 0;
```

## Storing and Decoding Bitmasks

This pattern is useful when a legacy schema stores permissions or feature flags as integers, and you need human-readable output in reports or APIs:

```sql
SELECT
  order_id,
  MAKE_SET(shipping_options, 'standard', 'express', 'overnight', 'signature_required') AS options
FROM orders
ORDER BY order_date DESC;
```

## Summary

`MAKE_SET()` converts a bitmask integer into a comma-separated string of labels. Each bit position maps to a positional string argument. NULL strings are skipped silently. It works well with `FIND_IN_SET()` for bitmask-based filtering and is particularly valuable when reading permission flags, feature toggles, or configuration bitmasks stored as integers in legacy schemas.
