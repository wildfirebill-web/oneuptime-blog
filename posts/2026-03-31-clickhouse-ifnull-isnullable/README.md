# How to Use ifNull() and isNullable() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, NULL Handling, Nullable, String Function, Query Pattern

Description: Learn how ifNull() substitutes a fallback value for NULLs and isNullable() checks whether a column type is Nullable in ClickHouse, enabling safe null-handling in queries.

---

`ifNull(value, fallback)` returns `value` if it is not NULL, otherwise returns `fallback`. It is the ClickHouse equivalent of `COALESCE(value, fallback)` with exactly two arguments, and it works with any `Nullable(T)` column. `isNullable(x)` is a type-level predicate that returns `1` if the expression or column is of `Nullable(T)` type, and `0` otherwise. It is evaluated at query planning time and is useful for introspecting schema types and writing generic macros.

## Creating a Table with Nullable Columns

```sql
CREATE TABLE user_profiles
(
    user_id     UInt64,
    username    String,
    email       Nullable(String),
    age         Nullable(UInt8),
    country     Nullable(String),
    score       Nullable(Float64)
)
ENGINE = MergeTree()
ORDER BY user_id;
```

```sql
INSERT INTO user_profiles VALUES
    (1, 'alice',   'alice@example.com', 30,   'US', 9.5),
    (2, 'bob',     NULL,               25,   'UK', NULL),
    (3, 'charlie', 'c@example.com',     NULL, NULL, 7.2),
    (4, 'diana',   NULL,               NULL, 'DE', NULL),
    (5, 'eve',     'eve@example.com',   22,   'FR', 8.8);
```

## Basic ifNull() Usage

```sql
-- Replace NULL email with a placeholder
SELECT
    user_id,
    username,
    ifNull(email,   'no-email@unknown.com') AS safe_email,
    ifNull(age,     0)                       AS safe_age,
    ifNull(country, 'Unknown')               AS safe_country,
    ifNull(score,   0.0)                     AS safe_score
FROM user_profiles
ORDER BY user_id;
```

```text
user_id  username  safe_email             safe_age  safe_country  safe_score
1        alice     alice@example.com      30        US            9.5
2        bob       no-email@unknown.com   25        UK            0.0
3        charlie   c@example.com          0         Unknown       7.2
4        diana     no-email@unknown.com   0         DE            0.0
5        eve       eve@example.com        22        FR            8.8
```

## ifNull() in Aggregations

NULL values are excluded from most aggregate functions. Use `ifNull` to substitute a default before aggregating.

```sql
-- Average score treating NULLs as 0 vs excluding NULLs
SELECT
    avg(score)               AS avg_excluding_nulls,  -- NULLs ignored
    avg(ifNull(score, 0.0))  AS avg_treating_null_as_0
FROM user_profiles;
```

```text
avg_excluding_nulls  avg_treating_null_as_0
8.5                  5.1
```

## ifNull() with Computed Expressions

The fallback can be any expression, not just a literal.

```sql
-- Use another column as the fallback
SELECT
    user_id,
    username,
    ifNull(email, concat(username, '@defaultdomain.com')) AS resolved_email
FROM user_profiles
ORDER BY user_id;
```

```text
user_id  username  resolved_email
1        alice     alice@example.com
2        bob       bob@defaultdomain.com
3        charlie   c@example.com
4        diana     diana@defaultdomain.com
5        eve       eve@example.com
```

## Chaining ifNull for Multiple Fallbacks

For more than two options, chain `ifNull` calls (or use `coalesce`).

```sql
-- Use email; fall back to username-based address; then to a generic placeholder
SELECT
    user_id,
    ifNull(
        email,
        ifNull(
            nullIf(username, ''),
            'anonymous@example.com'
        )
    ) AS best_email
FROM user_profiles
ORDER BY user_id;
```

## isNullable() - Type Inspection

`isNullable(expr)` returns `1` at query time if the expression type is `Nullable(T)`, and `0` if it is a non-nullable type.

```sql
-- Check which columns are Nullable
SELECT
    isNullable(user_id)  AS user_id_nullable,  -- UInt64, not nullable
    isNullable(email)    AS email_nullable,      -- Nullable(String)
    isNullable(age)      AS age_nullable,        -- Nullable(UInt8)
    isNullable(username) AS username_nullable    -- String, not nullable
FROM user_profiles
LIMIT 1;
```

```text
user_id_nullable  email_nullable  age_nullable  username_nullable
0                 1               1             0
```

Note: `isNullable` returns the same value for every row because it reflects the column type, not the row value. To check whether a specific value is NULL, use `isNull(expr)`.

## isNullable() vs isNull()

```sql
-- isNullable: column type check (same for every row)
-- isNull:     runtime value check (per-row)
SELECT
    user_id,
    isNullable(email)  AS email_type_is_nullable,  -- always 1 for this column
    isNull(email)      AS this_row_email_is_null    -- per-row check
FROM user_profiles
ORDER BY user_id;
```

```text
user_id  email_type_is_nullable  this_row_email_is_null
1        1                       0
2        1                       1
3        1                       0
4        1                       1
5        1                       0
```

## Using ifNull in ORDER BY

```sql
-- Sort users by score, treating NULL as -infinity (sort nulls last manually)
SELECT
    user_id,
    username,
    score
FROM user_profiles
ORDER BY ifNull(score, -1e10) DESC;
```

```text
user_id  username  score
1        alice     9.5
5        eve       8.8
3        charlie   7.2
2        bob       NULL
4        diana     NULL
```

## Conditional Logic with ifNull

```sql
-- Categorize users based on nullable score
SELECT
    user_id,
    username,
    score,
    multiIf(
        isNull(score),         'No score',
        ifNull(score, 0) >= 9, 'Excellent',
        ifNull(score, 0) >= 7, 'Good',
        'Needs improvement'
    ) AS rating
FROM user_profiles
ORDER BY user_id;
```

```text
user_id  username  score  rating
1        alice     9.5    Excellent
2        bob       NULL   No score
3        charlie   7.2    Good
4        diana     NULL   No score
5        eve       8.8    Good
```

## Summary

`ifNull(value, fallback)` substitutes `fallback` whenever `value` is NULL, making it essential for safe aggregations, display formatting, and downstream joins that cannot handle NULL inputs. `isNullable(expr)` is a type-level check that returns `1` for `Nullable(T)` column types and is evaluated once per query, not per row. For per-row NULL checks use `isNull(expr)`. When you need to handle more than one possible fallback, either chain `ifNull` calls or use `coalesce`.
