# How to Use assumeNotNull() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, NULL Handling, Performance, Type Conversion, Optimization

Description: Learn how to use assumeNotNull() in ClickHouse to strip the Nullable wrapper and improve query performance when you know a column has no NULLs.

---

`assumeNotNull(x)` converts a `Nullable(T)` column to `T`, asserting that the column contains no NULL values. This strips the nullable wrapper and allows the query optimizer to skip null checks, which can improve performance. It is a "trust me" instruction to the database - if the column actually contains NULLs, the behavior is undefined and may produce incorrect results. Use it only when you can guarantee at the data level that NULLs do not exist.

## Basic Usage

```sql
-- Convert Nullable(String) to String
SELECT assumeNotNull(nullable_column) AS non_nullable
FROM my_table
LIMIT 5;

-- Check the type difference
SELECT
    toTypeName(nullable_column)              AS original_type,
    toTypeName(assumeNotNull(nullable_column)) AS assumed_type
FROM my_table
LIMIT 1;
```

## When to Use assumeNotNull

Use `assumeNotNull` when:
1. You have verified the column has no NULLs in practice (even if the schema allows them)
2. A function requires a non-nullable input type
3. You want to eliminate the overhead of NULL checks in a hot query path

```sql
-- Verify there are no NULLs before using assumeNotNull
SELECT count() AS null_count
FROM my_table
WHERE nullable_column IS NULL;
-- If this returns 0, it is safe to use assumeNotNull
```

## Combining Nullable and Non-Nullable Columns

Some ClickHouse operations require all inputs to be the same type. If you are combining a `Nullable(String)` and a `String`, you need to either make both nullable with `toNullable()` or strip the nullable wrapper with `assumeNotNull()`.

```sql
-- Suppose col_a is Nullable(String) and col_b is String
-- Without assumeNotNull this could fail due to type mismatch
SELECT
    concat(assumeNotNull(col_a), ' - ', col_b) AS combined
FROM my_table
WHERE col_a IS NOT NULL
LIMIT 10;
```

## Using assumeNotNull in Array Operations

Array functions often require non-nullable element types. `assumeNotNull` is useful when building arrays from nullable columns.

```sql
-- Build an array from a nullable column (after verifying no NULLs exist)
SELECT
    user_id,
    groupArray(assumeNotNull(tag)) AS tags
FROM user_tags
WHERE tag IS NOT NULL
GROUP BY user_id
LIMIT 10;
```

## Improving Aggregation Performance

On very large tables, stripping the nullable wrapper can reduce per-row overhead during aggregation.

```sql
-- Both are correct if no NULLs exist, but assumeNotNull may be faster
-- on large tables with many rows
SELECT
    avg(assumeNotNull(response_time_ms)) AS avg_response_ms,
    max(assumeNotNull(response_time_ms)) AS max_response_ms
FROM api_metrics
WHERE response_time_ms IS NOT NULL;
```

## The Risk: Undefined Behavior on NULL Input

The behavior when NULLs are present is explicitly undefined - it may return garbage values, 0, or corrupt other results.

```sql
-- DANGEROUS: Only use this if you are certain there are no NULLs
-- If any row has NULL in value, the result is undefined
SELECT assumeNotNull(value) AS risky_column
FROM my_table;

-- SAFE: Filter NULLs first, then use assumeNotNull in a subquery
SELECT assumeNotNull(value) AS safe_column
FROM my_table
WHERE value IS NOT NULL;
```

## Checking Column Nullability

Before using `assumeNotNull`, always check whether your column actually contains NULLs:

```sql
-- Check for NULLs in a column before deciding whether to use assumeNotNull
SELECT
    'value_column'          AS column_name,
    count()                 AS total_rows,
    countIf(value IS NULL)  AS null_count,
    countIf(value IS NULL) = 0 AS safe_to_use_assumeNotNull
FROM my_table;
```

## Summary

`assumeNotNull(x)` converts a `Nullable(T)` to `T` by asserting no NULLs exist. It allows the optimizer to skip null checks and is useful when combining nullable and non-nullable columns in expressions or array operations. The critical caveat is that if the column does contain NULLs, behavior is undefined. Always verify with a null count query before using it, or pair it with a `WHERE col IS NOT NULL` filter to guarantee safety.
