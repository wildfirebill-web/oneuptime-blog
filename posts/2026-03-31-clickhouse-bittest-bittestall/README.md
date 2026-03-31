# How to Use bitTest() and bitTestAll() Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Bitwise, bitTest, bitTestAll, bitTestAny, Bitmask, Flag

Description: Learn how to use bitTest(), bitTestAll(), and bitTestAny() in ClickHouse to check individual bits or multiple bits in integer values for flag-based filtering.

---

ClickHouse provides `bitTest()`, `bitTestAll()`, and `bitTestAny()` for inspecting specific bits within integer values. These functions are essential when columns store multiple boolean flags packed into a single integer.

## bitTest(n, pos)

`bitTest(n, pos)` returns 1 if bit at position `pos` (0-indexed from the least significant bit) is set in `n`, and 0 otherwise:

```sql
SELECT
    bitTest(13, 0) AS bit0,  -- 13 = 1101b, bit 0 = 1
    bitTest(13, 1) AS bit1,  -- bit 1 = 0
    bitTest(13, 2) AS bit2,  -- bit 2 = 1
    bitTest(13, 3) AS bit3;  -- bit 3 = 1
```

```text
bit0 | bit1 | bit2 | bit3
-----+------+------+-----
1    | 0    | 1    | 1
```

## bitTestAll(n, pos1, pos2, ...)

`bitTestAll(n, pos1, pos2, ...)` returns 1 only if ALL listed bit positions are set:

```sql
SELECT bitTestAll(13, 0, 2, 3) AS all_three_set;  -- 1 (all set in 1101b)
SELECT bitTestAll(13, 0, 1, 2) AS all_three_set;  -- 0 (bit 1 is NOT set)
```

## bitTestAny(n, pos1, pos2, ...)

`bitTestAny(n, pos1, pos2, ...)` returns 1 if ANY of the listed bit positions are set:

```sql
SELECT bitTestAny(13, 1, 4, 5) AS any_set;  -- 0 (none of these bits are set in 1101b)
SELECT bitTestAny(13, 1, 2, 4) AS any_set;  -- 1 (bit 2 is set)
```

## Filtering Rows by Feature Flags

Suppose a `feature_flags` column stores which product features are enabled per user:

```sql
-- Bit positions for features:
-- 0 = dark_mode
-- 1 = beta_access
-- 2 = advanced_search
-- 3 = export_csv

SELECT user_id
FROM users
WHERE bitTest(feature_flags, 1) = 1  -- has beta access
  AND bitTest(feature_flags, 3) = 1; -- and can export CSV
```

## Using bitTestAll() for Role Requirements

A role requires specific permissions. `bitTestAll` checks all at once:

```sql
SELECT
    user_id,
    role,
    bitTestAll(permissions, 0, 2, 4) AS has_required_permissions
FROM user_roles
WHERE bitTestAll(permissions, 0, 2, 4) = 1;
```

## Aggregating Flag Statistics

Count how many users have each feature enabled:

```sql
SELECT
    'dark_mode'       AS feature, countIf(bitTest(feature_flags, 0) = 1) AS users FROM users
UNION ALL
SELECT 'beta_access',             countIf(bitTest(feature_flags, 1) = 1) FROM users
UNION ALL
SELECT 'advanced_search',         countIf(bitTest(feature_flags, 2) = 1) FROM users
UNION ALL
SELECT 'export_csv',              countIf(bitTest(feature_flags, 3) = 1) FROM users;
```

## Comparing bitTest() vs. Bitwise AND

`bitTest(n, pos)` is equivalent to `(n >> pos) & 1` and to `n & (1 << pos) > 0`:

```sql
-- These three are equivalent:
SELECT bitTest(13, 2)           AS method1,
       (13 >> 2) & 1             AS method2,
       (13 & (1 << 2)) > 0       AS method3;
```

`bitTest()` is more readable; use it whenever clarity matters.

## Summary

`bitTest(n, pos)` checks a single bit at a given position. `bitTestAll()` requires all specified bits to be set, and `bitTestAny()` requires at least one. These functions are cleaner and more readable than raw bitwise AND expressions, making them the preferred choice for flag-based filtering and feature-flag analysis in ClickHouse.
