# How to Use bitTest() and bitTestAll() Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Bitwise, bitTest, bitTestAll, Bit Operation, Flag

Description: Learn how to use bitTest() and bitTestAll() in ClickHouse to check individual bit positions and test whether all specified bits are set in an integer.

---

ClickHouse provides `bitTest()` and `bitTestAll()` for testing specific bit positions in integer values. These functions are essential when working with packed flag fields, permissions, and status codes stored as integers.

## Function Signatures

```sql
bitTest(num, pos)              -> UInt8
bitTestAll(num, pos1, pos2, ...) -> UInt8
```

- `bitTest(num, pos)` returns 1 if the bit at position `pos` (0-indexed from the least significant bit) is set, 0 otherwise.
- `bitTestAll(num, pos1, pos2, ...)` returns 1 only if ALL specified bit positions are set.

## Basic bitTest() Example

```sql
SELECT
    bitTest(13, 0) AS bit0,
    bitTest(13, 1) AS bit1,
    bitTest(13, 2) AS bit2,
    bitTest(13, 3) AS bit3
```

```text
bit0  bit1  bit2  bit3
----  ----  ----  ----
1     0     1     1
```

13 in binary is `1101`, so bits 0, 2, and 3 are set.

## Testing Permission Flags

Define permission bit positions and test them individually:

```sql
-- Bit positions: 0=READ, 1=WRITE, 2=DELETE, 3=ADMIN
SELECT
    user_id,
    permissions,
    bitTest(permissions, 0) AS can_read,
    bitTest(permissions, 1) AS can_write,
    bitTest(permissions, 2) AS can_delete,
    bitTest(permissions, 3) AS is_admin
FROM user_permissions
ORDER BY user_id
LIMIT 10
```

## Using bitTestAll() for Multi-Bit Check

Check if a user has both WRITE and DELETE permissions:

```sql
SELECT
    user_id,
    bitTestAll(permissions, 1, 2) AS can_write_and_delete
FROM user_permissions
WHERE can_write_and_delete = 1
```

`bitTestAll()` is equivalent to checking each bit individually with AND, but more concise.

## Filtering by Feature Flags

```sql
SELECT
    user_id,
    feature_flags
FROM user_features
WHERE bitTest(feature_flags, 5) = 1  -- feature at bit 5 is enabled
```

## Comparing bitTest() vs bitmaskToArray()

`bitTest()` is more efficient when checking one or two specific bit positions. `bitmaskToArray()` is better when you need to inspect all set bits or iterate over them.

```sql
-- Single bit check - use bitTest():
SELECT bitTest(flags, 3) AS is_premium FROM users;

-- All bits check - use bitmaskToArray():
SELECT bitmaskToArray(flags) AS all_flags FROM users;
```

## Aggregating Bit Flag Statistics

```sql
SELECT
    countIf(bitTest(status_flags, 0) = 1) AS flagged_count,
    countIf(bitTest(status_flags, 1) = 1) AS reviewed_count,
    countIf(bitTest(status_flags, 2) = 1) AS approved_count,
    count() AS total
FROM submissions
WHERE submission_date >= today() - 7
```

## Bit Position Reference

Bit positions are 0-indexed from the right (least significant bit):

```text
Integer: 13
Binary:  1101
         3210  <-- bit positions
```

Position 0 = value 1, position 1 = value 2, position 2 = value 4, position 3 = value 8.

## Summary

`bitTest()` checks whether a single bit is set in an integer, while `bitTestAll()` checks whether multiple bits are all set at once. These functions are the most direct way to query packed bit flags in ClickHouse, making them ideal for permission systems, feature toggles, and status codes stored as integers.
