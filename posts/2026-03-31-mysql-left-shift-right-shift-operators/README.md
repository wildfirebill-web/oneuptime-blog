# How to Use Left Shift (<<) and Right Shift (>>) Operators in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Bitwise, Operator, Integer

Description: Learn how to use the Left Shift (<<) and Right Shift (>>) operators in MySQL to manipulate integers at the bit level and build bitmask constants efficiently.

---

## What Are the Shift Operators?

MySQL provides two bitwise shift operators:

- `<<` (left shift) - moves all bits to the left by N positions, effectively multiplying by powers of 2
- `>>` (right shift) - moves all bits to the right by N positions, effectively dividing by powers of 2

Both operators treat operands as 64-bit unsigned integers and discard bits that fall off either end.

## Basic Syntax

```sql
expr << N   -- Left shift by N bits
expr >> N   -- Right shift by N bits
```

## Simple Examples

```sql
-- Left shift: multiply by powers of 2
SELECT 1 << 0;   -- Result: 1
SELECT 1 << 1;   -- Result: 2
SELECT 1 << 2;   -- Result: 4
SELECT 1 << 3;   -- Result: 8
SELECT 1 << 8;   -- Result: 256

-- Right shift: integer division by powers of 2
SELECT 256 >> 1;  -- Result: 128
SELECT 256 >> 4;  -- Result: 16
SELECT 7 >> 1;    -- Result: 3 (bits shifted out are discarded)
```

## Building Bitmask Constants

Left shift is the cleanest way to define bit flag constants in queries:

```sql
-- Define flags using shifts instead of magic numbers
-- Bit 0 = 1<<0 = 1 (READ)
-- Bit 1 = 1<<1 = 2 (WRITE)
-- Bit 2 = 1<<2 = 4 (DELETE)
-- Bit 3 = 1<<3 = 8 (ADMIN)

-- Check if bit 3 (ADMIN) is set
SELECT name, permissions
FROM users
WHERE (permissions & (1 << 3)) != 0;
```

## Shifting in Calculations

```sql
-- Compute 2^10 using left shift
SELECT 1 << 10;  -- Result: 1024

-- Extract the Nth bit of a value
-- Shift right N positions, then AND with 1
SELECT (permissions >> 2) & 1 AS has_delete_flag
FROM users;
```

## Extracting Nibbles and Bytes

```sql
-- Extract the high byte (bits 8-15) from a 16-bit value
SELECT (value >> 8) & 255 AS high_byte,
       value & 255 AS low_byte
FROM packed_data;

-- Extract the upper 4 bits (nibble) from a byte
SELECT (byte_val >> 4) AS upper_nibble,
       byte_val & 15 AS lower_nibble
FROM sensor_readings;
```

## Generating a Bitmask for N Bits

```sql
-- Create a mask for the lower N bits
-- Lower 4 bits: ~(~0 << 4) = 15
SELECT ~(~0 << 4);  -- Result: 15  (0000...1111)
SELECT ~(~0 << 8);  -- Result: 255 (0000...11111111)

-- Apply to isolate lower bits
SELECT value & ~(~0 << 4) AS lower_4_bits
FROM packed_values;
```

## Range Considerations

```sql
-- Left shifting beyond 63 positions results in 0
SELECT 1 << 64;   -- Result: 0 (overflows 64-bit range)

-- Right shifting always terminates at 0
SELECT 1 >> 64;   -- Result: 0

-- NULL propagates
SELECT NULL << 3;  -- Result: NULL
SELECT 8 >> NULL;  -- Result: NULL
```

## Combining with Other Bitwise Operators

```sql
-- Set bits 2 and 5 using shifts and OR
UPDATE settings
SET flags = flags | (1 << 2) | (1 << 5)
WHERE user_id = 10;

-- Clear bit 3 using shift and AND NOT
UPDATE settings
SET flags = flags & ~(1 << 3)
WHERE user_id = 10;
```

## Summary

The left shift (`<<`) and right shift (`>>`) operators in MySQL make working with bitmasks readable and self-documenting. Instead of hardcoding values like `8` or `16`, use `1 << 3` or `1 << 4` to clearly express which bit you are targeting. Combine them with `&`, `|`, and `~` for a complete toolkit to manage permission flags, feature toggles, and packed integer data in a MySQL database.
