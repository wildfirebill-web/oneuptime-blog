# How to Use Bool Data Type in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Data Types, Bool, Boolean

Description: Learn how ClickHouse's Bool type works, its relationship to UInt8, true/false literals, storage, and practical usage in queries.

---

ClickHouse introduced a native `Bool` data type to represent boolean values cleanly. Before this, developers relied on `UInt8` with values `0` and `1` to simulate booleans. Understanding how `Bool` works - and how it relates to `UInt8` - helps you write cleaner schemas and avoid subtle bugs when filtering or comparing boolean columns.

## What is the Bool Data Type

The `Bool` type in ClickHouse stores boolean values as `true` or `false`. Internally it is stored as a single byte (identical to `UInt8`), but it is displayed as `true` or `false` rather than `1` or `0`. This makes query results and schema definitions far more readable.

```sql
CREATE TABLE user_flags
(
    user_id     UInt64,
    is_active   Bool,
    is_verified Bool,
    is_premium  Bool
)
ENGINE = MergeTree()
ORDER BY user_id;
```

## Inserting Bool Values

You can insert boolean values using the literals `true` and `false` (case-insensitive), or using the numeric equivalents `1` and `0`. ClickHouse accepts both forms interchangeably.

```sql
INSERT INTO user_flags VALUES
    (1, true,  true,  false),
    (2, false, true,  true),
    (3, true,  false, false),
    (4, 1,     0,     1);
```

Verify the inserted data:

```sql
SELECT *
FROM user_flags;
```

Output will display `true` and `false` rather than `1` and `0` for Bool columns.

## Bool and UInt8 Compatibility

`Bool` is an alias for `UInt8` in terms of storage, so numeric comparisons work seamlessly. You can compare a `Bool` column to `1` or `0` as well as `true` or `false`.

```sql
-- All three queries are equivalent
SELECT * FROM user_flags WHERE is_active = true;
SELECT * FROM user_flags WHERE is_active = 1;
SELECT * FROM user_flags WHERE is_active != 0;
```

Because `Bool` maps to `UInt8`, arithmetic operations also work, though they are rarely semantically useful:

```sql
SELECT
    user_id,
    is_active + is_verified AS flag_count
FROM user_flags;
```

## Casting and Type Conversions

You can cast other types to `Bool` using `CAST` or `toBool`. Non-zero numeric values become `true`, and zero becomes `false`. Non-empty strings that represent truthy values also cast accordingly.

```sql
SELECT
    CAST(1 AS Bool)   AS from_one,
    CAST(0 AS Bool)   AS from_zero,
    CAST(42 AS Bool)  AS from_nonzero,
    toBool(1)         AS using_function;
```

Casting from strings:

```sql
SELECT
    CAST('true'  AS Bool) AS str_true,
    CAST('false' AS Bool) AS str_false,
    CAST('1'     AS Bool) AS str_one,
    CAST('0'     AS Bool) AS str_zero;
```

## Using Bool in WHERE Clauses

Bool columns work naturally in `WHERE` clauses without needing explicit comparison operators:

```sql
-- Shorthand - works because Bool is UInt8 (truthy when non-zero)
SELECT user_id
FROM user_flags
WHERE is_active;

-- Explicit comparison - preferred for clarity
SELECT user_id
FROM user_flags
WHERE is_active = true;

-- Negation
SELECT user_id
FROM user_flags
WHERE NOT is_active;

-- Combining conditions
SELECT user_id
FROM user_flags
WHERE is_active AND is_verified AND NOT is_premium;
```

## Bool in Aggregations

Bool columns can be used with aggregate functions for counting or summing flag values:

```sql
SELECT
    countIf(is_active)                       AS active_users,
    countIf(is_verified)                     AS verified_users,
    countIf(is_active AND is_verified)        AS active_and_verified,
    sum(is_premium)                          AS premium_count,
    avg(is_active)                           AS active_ratio
FROM user_flags;
```

## Nullable Bool

You can wrap `Bool` in `Nullable` to allow unknown boolean states, but do this sparingly since Nullable carries overhead:

```sql
CREATE TABLE user_consents
(
    user_id         UInt64,
    email_consent   Nullable(Bool),
    sms_consent     Nullable(Bool)
)
ENGINE = MergeTree()
ORDER BY user_id;

INSERT INTO user_consents VALUES
    (1, true,  NULL),
    (2, NULL,  false),
    (3, true,  true);

SELECT
    user_id,
    isNull(email_consent) AS email_unknown,
    ifNull(sms_consent, false) AS sms_resolved
FROM user_consents;
```

## Bool in System Tables

ClickHouse's own system tables use `Bool` columns extensively. You can explore them to see Bool in practice:

```sql
SELECT
    name,
    engine,
    is_temporary
FROM system.tables
WHERE is_temporary = false
LIMIT 10;

SELECT
    name,
    type,
    is_in_primary_key,
    is_in_sorting_key
FROM system.columns
WHERE table = 'user_flags';
```

## Storage and Performance

Since `Bool` is backed by `UInt8`, it occupies 1 byte per value. For large tables with many boolean flags, consider packing multiple booleans into a single `UInt8` or `UInt64` using bitwise operations if storage is critical. However, for clarity and maintainability, individual `Bool` columns are usually preferred.

```sql
-- Storage size inspection
SELECT
    column,
    sum(data_compressed_bytes)   AS compressed_bytes,
    sum(data_uncompressed_bytes) AS uncompressed_bytes
FROM system.parts_columns
WHERE table = 'user_flags'
GROUP BY column
ORDER BY column;
```

## Summary

ClickHouse's `Bool` type provides a clean, readable way to store boolean values backed by the efficient `UInt8` storage format. It accepts `true`/`false` literals as well as `1`/`0`, making it backward-compatible with older UInt8-based boolean patterns. Use `Bool` for flag columns to improve schema clarity and query readability.
