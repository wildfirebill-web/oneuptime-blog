# How to Use toTypeName() to Check Column Types in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, toTypeName, Type Inspection, Schema, Sql

Description: Learn how to use toTypeName() in ClickHouse to inspect the data type of any expression or column at query time for debugging and schema validation.

---

## Overview

`toTypeName(value)` returns a String describing the ClickHouse data type of the provided expression. It is invaluable during development for understanding inferred types, debugging type mismatch errors, and verifying schema assumptions.

## Basic Usage

```sql
SELECT
    toTypeName(1)                AS int_type,
    toTypeName(1.5)              AS float_type,
    toTypeName('hello')          AS string_type,
    toTypeName(now())            AS datetime_type,
    toTypeName(today())          AS date_type
```

Output:

```text
int_type | float_type | string_type | datetime_type | date_type
UInt8    | Float64    | String      | DateTime      | Date
```

## Checking Nullable Types

```sql
SELECT
    toTypeName(NULL)                      AS null_type,
    toTypeName(toNullable(42))            AS nullable_int,
    toTypeName(assumeNotNull(toNullable(1))) AS not_null
```

Output:

```text
null_type | nullable_int   | not_null
Nullable(Nothing) | Nullable(UInt8) | UInt8
```

## Inspecting Array and Map Types

```sql
SELECT
    toTypeName([1, 2, 3])                      AS array_type,
    toTypeName(map('a', 1))                    AS map_type,
    toTypeName(tuple('x', 42, 3.14))           AS tuple_type
```

Output:

```text
array_type   | map_type            | tuple_type
Array(UInt8) | Map(String, UInt8)  | Tuple(String, UInt8, Float64)
```

## Verifying Column Types in Tables

Use `toTypeName` on actual columns to confirm the type at query time:

```sql
SELECT
    event_id,
    toTypeName(event_id)   AS event_id_type,
    toTypeName(event_time) AS event_time_type,
    toTypeName(metadata)   AS metadata_type
FROM events
LIMIT 1
```

You can also check schema via the system tables:

```sql
SELECT name, type
FROM system.columns
WHERE table = 'events' AND database = currentDatabase()
```

## Debugging Type Mismatch Errors

When you encounter a type error in a JOIN or comparison, wrap both sides in `toTypeName()` to diagnose the issue:

```sql
SELECT
    toTypeName(a.user_id) AS left_type,
    toTypeName(b.user_id) AS right_type
FROM table_a a
CROSS JOIN table_b b
LIMIT 1
```

If the types differ (e.g., `UInt64` vs `Int64`), you need an explicit `CAST`.

## Type Detection in Dynamic Queries

Use `toTypeName` with conditional logic to branch based on type:

```sql
SELECT
    col,
    CASE
        WHEN toTypeName(col) LIKE 'Nullable%' THEN 'nullable'
        WHEN toTypeName(col) LIKE 'Array%'    THEN 'array'
        ELSE 'scalar'
    END AS col_category
FROM my_table
LIMIT 10
```

## Checking LowCardinality Wrapping

```sql
SELECT toTypeName(toLowCardinality('example')) AS lc_type
-- LowCardinality(String)
```

This helps confirm whether a column is using `LowCardinality` optimization.

## Fixed String and UUID Types

```sql
SELECT
    toTypeName(toFixedString('hello', 10)) AS fixed_str_type,
    toTypeName(toUUID('6ba7b810-9dad-11d1-80b4-00c04fd430c8')) AS uuid_type
```

Output:

```text
fixed_str_type | uuid_type
FixedString(10) | UUID
```

## Summary

`toTypeName()` returns the ClickHouse type name of any expression as a String. It is an essential debugging tool for understanding inferred types during development, diagnosing type mismatches in JOINs, and validating schema assumptions. Combined with `system.columns`, it provides complete runtime type introspection.
