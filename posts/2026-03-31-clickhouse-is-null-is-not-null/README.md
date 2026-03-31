# How to Use IS NULL and IS NOT NULL in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, NULL, IS NULL, IS NOT NULL, Nullable

Description: Understand how IS NULL and IS NOT NULL work in ClickHouse with Nullable columns, and learn safe patterns for filtering and aggregating nullable data.

---

## Nullable Types in ClickHouse

ClickHouse columns are not nullable by default. To store NULL values, a column must be declared as `Nullable(T)`, for example `Nullable(String)` or `Nullable(Float64)`. Non-nullable columns can never be NULL.

```sql
CREATE TABLE users (
  user_id UInt64,
  email String,
  phone Nullable(String),
  score Nullable(Float64)
) ENGINE = MergeTree()
ORDER BY user_id
```

## IS NULL and IS NOT NULL

Use `IS NULL` to find rows where a Nullable column holds no value, and `IS NOT NULL` to find rows with a value present.

```sql
-- Users with no phone number
SELECT user_id, email
FROM users
WHERE phone IS NULL

-- Users who have a score
SELECT user_id, score
FROM users
WHERE score IS NOT NULL
ORDER BY score DESC
```

## Why = NULL Does Not Work

Comparing with `= NULL` always returns `NULL`, not `1` or `0`, because NULL represents an unknown value.

```sql
-- This returns 0 rows - always false
SELECT * FROM users WHERE phone = NULL;

-- This is correct
SELECT * FROM users WHERE phone IS NULL;
```

## isNull and isNotNull Functions

ClickHouse also provides `isNull(x)` and `isNotNull(x)` function forms that return `UInt8`.

```sql
SELECT
  user_id,
  isNull(phone) AS has_no_phone,
  isNotNull(score) AS has_score
FROM users
```

These are useful when composing expressions or using NULL checks inside `arrayFilter`.

## Filtering in Aggregations

Aggregate functions skip NULL values automatically. To count NULLs explicitly, use `countIf(isNull(col))`.

```sql
SELECT
  count() AS total_users,
  countIf(isNull(phone)) AS users_without_phone,
  countIf(isNotNull(score)) AS scored_users,
  avg(score) AS avg_score  -- ignores NULLs
FROM users
```

## Coalesce for Default Values

Use `coalesce` to replace NULLs with a fallback value.

```sql
SELECT
  user_id,
  coalesce(phone, 'N/A') AS phone_display,
  coalesce(score, 0.0) AS score_with_default
FROM users
```

## ifNull as a Shorthand

`ifNull(x, default)` is equivalent to `coalesce(x, default)` for a single column.

```sql
SELECT ifNull(score, 0) AS safe_score FROM users
```

## Summary

Use `IS NULL` and `IS NOT NULL` (or their function forms `isNull` / `isNotNull`) to filter nullable columns in ClickHouse. Never use `= NULL` - it always evaluates to NULL. Aggregate functions automatically skip NULLs; use `coalesce` or `ifNull` to substitute defaults in SELECT expressions.
