# How to Use BIN() and HEX() Functions for Number Conversion in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, BIN Function, HEX Function, Number Conversion, SQL Functions

Description: Learn how to use BIN() and HEX() functions in MySQL to convert numbers to binary and hexadecimal representations with practical examples.

---

## What Are BIN() and HEX()

MySQL provides two functions for converting integers to alternative number system representations:

- `BIN(n)` - converts an integer to its binary (base-2) string representation
- `HEX(n)` - converts an integer (or string) to its hexadecimal (base-16) string representation

Both return string representations, not numeric values. The complementary functions for converting back are:
- `CONV(str, 2, 10)` - binary string to decimal
- `UNHEX(str)` - hex string to binary string

## BIN() Function

```sql
BIN(number)
```

### Basic Usage

```sql
SELECT BIN(0);    -- Output: 0
SELECT BIN(1);    -- Output: 1
SELECT BIN(2);    -- Output: 10
SELECT BIN(5);    -- Output: 101
SELECT BIN(10);   -- Output: 1010
SELECT BIN(255);  -- Output: 11111111
SELECT BIN(1024); -- Output: 10000000000
```

### BIN() in Queries

```sql
SELECT
  id,
  permissions,
  BIN(permissions) AS binary_permissions
FROM users;
```

Useful for displaying bitmask values in binary for debugging:

```sql
SELECT
  user_id,
  permission_flags,
  BIN(permission_flags) AS flags_binary
FROM user_permissions
ORDER BY user_id;
```

## HEX() Function

```sql
HEX(number_or_string)
```

### HEX() with Numbers

```sql
SELECT HEX(0);      -- Output: 0
SELECT HEX(10);     -- Output: A
SELECT HEX(16);     -- Output: 10
SELECT HEX(255);    -- Output: FF
SELECT HEX(256);    -- Output: 100
SELECT HEX(65535);  -- Output: FFFF
```

### HEX() with Strings

When passed a string, `HEX()` returns the hexadecimal encoding of each character's byte values:

```sql
SELECT HEX('hello');  -- Output: 68656C6C6F
SELECT HEX('A');      -- Output: 41
SELECT HEX('MySQL');  -- Output: 4D7953514C
```

### HEX() for Binary Data

```sql
-- View binary column content as hex
SELECT HEX(binary_column) FROM blob_table;

-- Useful for UUID stored as BINARY(16)
SELECT BIN_TO_UUID(id) AS uuid, HEX(id) AS hex_id FROM sessions;
```

## CONV() - Universal Base Conversion

`CONV()` converts between any two bases (2-36):

```sql
SELECT CONV('1010', 2, 10);  -- Binary to decimal: Output 10
SELECT CONV('FF', 16, 10);   -- Hex to decimal: Output 255
SELECT CONV('10', 10, 2);    -- Decimal to binary: Output 1010
SELECT CONV('10', 10, 16);   -- Decimal to hex: Output A
SELECT CONV('777', 8, 10);   -- Octal to decimal: Output 511
```

## UNHEX() - Hex String to Binary

`UNHEX()` is the inverse of `HEX()` for strings:

```sql
SELECT UNHEX('48656C6C6F');  -- Output: Hello
SELECT UNHEX('FF');          -- Output: binary byte 0xFF
```

## Practical Use Case - Bitmask Permissions

```sql
-- Define permissions as bit flags
-- READ=1, WRITE=2, DELETE=4, ADMIN=8

CREATE TABLE user_perms (
  user_id INT PRIMARY KEY,
  flags TINYINT UNSIGNED DEFAULT 0
);

INSERT INTO user_perms VALUES (1, 5);  -- READ + DELETE (1 + 4 = 5)
INSERT INTO user_perms VALUES (2, 3);  -- READ + WRITE (1 + 2 = 3)
INSERT INTO user_perms VALUES (3, 15); -- All perms (1+2+4+8 = 15)

-- Display permissions in binary
SELECT
  user_id,
  flags,
  BIN(flags) AS binary_flags,
  flags & 1 AS can_read,
  flags & 2 AS can_write,
  flags & 4 AS can_delete,
  flags & 8 AS is_admin
FROM user_perms;
```

## Padding BIN() Output

BIN() does not zero-pad. To get fixed-width output:

```sql
-- Pad to 8 bits
SELECT LPAD(BIN(5), 8, '0') AS padded_binary;
-- Output: 00000101
```

## Summary

`BIN(n)` converts an integer to its binary string representation, and `HEX(n)` converts a number or string to hexadecimal. Use `BIN()` to display bitmask values for debugging, `HEX()` to inspect binary/BLOB column contents, and `CONV()` for conversions between arbitrary bases. The inverse functions are `CONV(str, 2, 10)` for binary-to-decimal and `UNHEX()` for hex-to-binary string conversion.
