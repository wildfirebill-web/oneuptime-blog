# How to Use toNullable() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Null Handling, Type Conversion, UNION ALL, Schema

Description: Learn how to use toNullable() in ClickHouse to convert non-nullable types to Nullable(T) for UNION ALL compatibility and nullable-aware functions.

---

`toNullable(x)` converts a non-nullable value or column to `Nullable(T)`. This is the reverse of `assumeNotNull()`. Its primary use is resolving type mismatches when combining nullable and non-nullable columns in `UNION ALL` queries, or when a function requires a `Nullable(T)` input but your column is a plain `T`.

## Basic Usage

```sql
-- Convert a non-nullable String to Nullable(String)
SELECT toNullable('hello') AS nullable_value;

-- Check the type transformation
SELECT
    toTypeName('hello')              AS original_type,
    toTypeName(toNullable('hello'))  AS nullable_type;
```

## Resolving UNION ALL Type Mismatches

The most common reason to use `toNullable()` is a `UNION ALL` between a table that has nullable columns and one that has non-nullable columns. ClickHouse requires compatible types across all branches of a `UNION ALL`.

```sql
-- This fails if archive_orders.amount is non-nullable
-- but recent_orders.amount is Nullable(Float64)

-- Fix: wrap the non-nullable column with toNullable()
SELECT
    order_id,
    amount,
    'recent' AS source
FROM recent_orders

UNION ALL

SELECT
    order_id,
    toNullable(amount) AS amount,  -- convert to match Nullable(Float64)
    'archive' AS source
FROM archive_orders

ORDER BY order_id
LIMIT 20;
```

## Combining Nullable and Non-Nullable in Expressions

Some expressions require all inputs to be the same nullable type. Use `toNullable()` to align them.

```sql
-- Suppose col_a is String (non-nullable) and col_b is Nullable(String)
-- coalesce requires compatible types
SELECT
    coalesce(toNullable(col_a), col_b) AS combined
FROM my_table
LIMIT 10;
```

## Using toNullable in Table Definitions

When inserting from a non-nullable source into a nullable column, the implicit conversion handles it. But when defining computed columns, `toNullable` makes the intent explicit.

```sql
CREATE TABLE nullable_demo
(
    id         UInt64,
    raw_value  UInt32,
    -- Explicitly nullable version for downstream joins
    null_value Nullable(UInt32) DEFAULT toNullable(raw_value)
)
ENGINE = MergeTree()
ORDER BY id;
```

## toNullable with Literals

Use `toNullable` with literals when you need a typed NULL or a nullable literal in an expression.

```sql
-- A nullable literal for use in conditional expressions
SELECT
    if(1 = 2, toNullable(42), NULL) AS conditional_null;
```

## Type Alignment for Array Functions

Some array functions that return nullable outputs require nullable inputs. Use `toNullable()` to prepare inputs.

```sql
-- Convert array elements to nullable for functions that require it
SELECT
    arrayMap(x -> toNullable(x), [1, 2, 3]) AS nullable_array;
```

## Checking Current Type

Always check the column type before using `toNullable()` - it is only needed when the column is currently non-nullable.

```sql
-- Check which columns in a table are already nullable
SELECT
    name,
    type,
    type LIKE 'Nullable%' AS is_nullable
FROM system.columns
WHERE table = 'my_table'
  AND database = currentDatabase()
ORDER BY name;
```

## toNullable vs Nullable Column Definition

```sql
-- At table definition time, declare nullable directly
CREATE TABLE example_a
(
    id    UInt64,
    value Nullable(String)   -- nullable from the start
)
ENGINE = MergeTree()
ORDER BY id;

-- At query time, convert a non-nullable to nullable
SELECT toNullable(non_nullable_col) AS converted
FROM example_a
LIMIT 5;
```

Declare columns as `Nullable(T)` in the schema when values are genuinely optional. Use `toNullable()` at query time only to resolve type conflicts in expressions or `UNION ALL`.

## Summary

`toNullable(x)` wraps a non-nullable type in `Nullable(T)`, enabling type-compatible operations with nullable columns. Its most common use is fixing `UNION ALL` type mismatches between queries where one source has nullable columns and another has non-nullable columns. It is the complement of `assumeNotNull()` - use `toNullable()` to widen a type (making it accept NULL), and `assumeNotNull()` to narrow it (asserting no NULLs exist).
