# How to Use CONV() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Conv Function, Base Conversion, Math Functions, Sql

Description: Learn how to use the CONV() function in MySQL to convert numbers between different numeric bases like binary, octal, decimal, and hexadecimal.

---

## Introduction

`CONV()` is a MySQL function that converts a number from one numeric base (radix) to another. It can handle bases from 2 to 36, covering binary (base 2), octal (base 8), decimal (base 10), hexadecimal (base 16), and custom bases. The function accepts and returns strings, as different bases use different character sets.

## Basic Syntax

```sql
CONV(number, from_base, to_base)
```

- `number`: the number to convert (as a string or integer).
- `from_base`: the base of the input number (2 to 36).
- `to_base`: the base to convert to (2 to 36).

Returns the converted value as a string, or NULL if any argument is NULL or invalid.

## Common Base Conversions

```sql
-- Decimal to Binary
SELECT CONV(255, 10, 2);   -- Returns: '11111111'
SELECT CONV(10, 10, 2);    -- Returns: '1010'

-- Binary to Decimal
SELECT CONV('11111111', 2, 10);  -- Returns: '255'
SELECT CONV('1010', 2, 10);      -- Returns: '10'

-- Decimal to Hexadecimal
SELECT CONV(255, 10, 16);  -- Returns: 'FF'
SELECT CONV(16, 10, 16);   -- Returns: '10'

-- Hexadecimal to Decimal
SELECT CONV('FF', 16, 10);  -- Returns: '255'
SELECT CONV('1A', 16, 10);  -- Returns: '26'

-- Decimal to Octal
SELECT CONV(8, 10, 8);    -- Returns: '10'
SELECT CONV(64, 10, 8);   -- Returns: '100'

-- Octal to Decimal
SELECT CONV('10', 8, 10); -- Returns: '8'
```

## Binary to Hexadecimal

```sql
SELECT CONV('11111111', 2, 16);  -- Returns: 'FF'
SELECT CONV('1010', 2, 16);      -- Returns: 'A'
```

## Hexadecimal to Binary

```sql
SELECT CONV('FF', 16, 2);   -- Returns: '11111111'
SELECT CONV('1A3', 16, 2);  -- Returns: '110100011'
```

## Converting IP Address Components

```sql
-- Convert each octet of an IP address to binary
SELECT
  CONV(192, 10, 2) AS octet1,
  CONV(168, 10, 2) AS octet2,
  CONV(1, 10, 2)   AS octet3,
  CONV(100, 10, 2) AS octet4;
```

## Storing Binary Flags

Convert a binary flag string to decimal for compact storage:

```sql
-- Permissions represented as binary: rwxr-xr-x = 755
SELECT CONV('111101101', 2, 10) AS decimal_permission;
-- Returns: '493'

-- Convert back
SELECT CONV(493, 10, 2);  -- Returns: '111101101'
SELECT CONV(493, 10, 8);  -- Returns: '755'
```

## Negative Numbers

Use a negative `to_base` to produce signed output:

```sql
SELECT CONV('FFFFFFFFFFFFFFFF', 16, -10);
-- Returns: -1 (interprets as signed 64-bit integer)

SELECT CONV('7FFFFFFFFFFFFFFF', 16, -10);
-- Returns: 9223372036854775807 (MAX INT64)
```

## Base 36 (Alphanumeric)

Base 36 uses digits 0-9 and letters A-Z, allowing compact representation of large numbers:

```sql
SELECT CONV(1000000, 10, 36);  -- Returns: 'LFLS' (compact encoding)
SELECT CONV('LFLS', 36, 10);   -- Returns: '1000000'
```

## CONV vs HEX and BIN Functions

MySQL also provides dedicated functions for hex and binary:

```sql
-- Decimal to hex
SELECT HEX(255);           -- Returns: 'FF'  (same as CONV(255, 10, 16))
SELECT CONV(255, 10, 16);  -- Returns: 'FF'

-- Decimal to binary
SELECT BIN(10);            -- Returns: '1010'  (same as CONV(10, 10, 2))
SELECT CONV(10, 10, 2);    -- Returns: '1010'

-- Hex to decimal
SELECT CONV('FF', 16, 10); -- Returns: '255'
-- No direct UNHEX to decimal; CONV is more flexible
```

## Practical Example: Color Code Conversion

Convert decimal RGB values to hex for CSS:

```sql
SELECT
  CONCAT(
    '#',
    LPAD(CONV(red, 10, 16), 2, '0'),
    LPAD(CONV(green, 10, 16), 2, '0'),
    LPAD(CONV(blue, 10, 16), 2, '0')
  ) AS hex_color
FROM product_colors;
-- Example: red=255, green=128, blue=0 => '#FF8000'
```

## Error Handling

```sql
SELECT CONV(NULL, 10, 2);   -- Returns: NULL
SELECT CONV('XYZ', 10, 2);  -- Returns: '0' (invalid input treated as 0)
SELECT CONV(10, 10, 1);     -- Returns: NULL (base must be 2-36)
```

## Summary

`CONV()` is MySQL's versatile number base conversion function, supporting bases from 2 to 36. Use it to convert between binary, octal, decimal, and hexadecimal representations. It returns the result as a string. For simple decimal-to-hex or decimal-to-binary conversions, the dedicated `HEX()` and `BIN()` functions are more concise, but `CONV()` handles any arbitrary base.
