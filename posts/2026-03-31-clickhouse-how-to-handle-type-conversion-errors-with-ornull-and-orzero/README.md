# How to Handle Type Conversion Errors with OrNull and OrZero in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Type Conversion, OrNull, OrZero, Error Handling, Sql

Description: Learn how to use OrNull and OrZero variants of ClickHouse type conversion functions to gracefully handle invalid input without query failures.

---

## Overview

ClickHouse conversion functions like `toInt32()`, `toFloat64()`, and `toDate()` throw an exception when given invalid input. The `OrNull` and `OrZero` variants provide safe alternatives - returning NULL or zero instead of failing. This is critical when parsing user-generated or external data.

## The Problem with Standard Conversions

```sql
SELECT toInt32('not_a_number') AS val
-- Exception: Cannot parse Int32 from String: Not a valid integer
```

This would cause the entire query to fail.

## OrNull Variants

Functions ending in `OrNull` return NULL for invalid input:

```sql
SELECT
    toInt32OrNull('42')          AS valid_int,
    toInt32OrNull('not_a_number') AS invalid_int,
    toFloat64OrNull('3.14')      AS valid_float,
    toFloat64OrNull('abc')       AS invalid_float
```

Output:

```text
valid_int | invalid_int | valid_float | invalid_float
42        | NULL        | 3.14        | NULL
```

## OrZero Variants

Functions ending in `OrZero` return 0 (or the zero value of the type) for invalid input:

```sql
SELECT
    toInt32OrZero('100')         AS valid_int,
    toInt32OrZero('bad_input')   AS invalid_int,
    toFloat64OrZero('2.71')      AS valid_float,
    toFloat64OrZero('xyz')       AS invalid_float
```

Output:

```text
valid_int | invalid_int | valid_float | invalid_float
100       | 0           | 2.71        | 0.0
```

## Complete List of Safe Conversion Functions

```sql
-- Integer variants
SELECT toInt8OrNull(x),   toInt8OrZero(x)
SELECT toInt16OrNull(x),  toInt16OrZero(x)
SELECT toInt32OrNull(x),  toInt32OrZero(x)
SELECT toInt64OrNull(x),  toInt64OrZero(x)
SELECT toUInt8OrNull(x),  toUInt8OrZero(x)
SELECT toUInt16OrNull(x), toUInt16OrZero(x)
SELECT toUInt32OrNull(x), toUInt32OrZero(x)
SELECT toUInt64OrNull(x), toUInt64OrZero(x)

-- Float variants
SELECT toFloat32OrNull(x), toFloat32OrZero(x)
SELECT toFloat64OrNull(x), toFloat64OrZero(x)

-- Date variants
SELECT toDateOrNull(x),     toDateOrZero(x)
SELECT toDateTimeOrNull(x), toDateTimeOrZero(x)

-- Decimal variant
SELECT toDecimal32OrNull(x, scale), toDecimal32OrZero(x, scale)
```

## Parsing External CSV Data

When loading CSV data with potentially malformed fields, safe conversions prevent import failures:

```sql
SELECT
    raw_user_id,
    toInt64OrNull(raw_user_id)      AS user_id,
    toFloat64OrNull(raw_revenue)    AS revenue,
    toDateOrNull(raw_event_date)    AS event_date
FROM raw_imports
WHERE toInt64OrNull(raw_user_id) IS NOT NULL
```

## Combining with ifNull() for Defaults

Use `ifNull()` to replace NULLs from `OrNull` conversions with a meaningful default:

```sql
SELECT
    raw_score,
    ifNull(toFloat64OrNull(raw_score), 0.0) AS score_with_default
FROM user_responses
```

## Filtering Invalid Rows

Use `OrNull` to identify and exclude rows with bad data:

```sql
SELECT count() AS total_rows,
       countIf(toInt64OrNull(user_id_str) IS NULL) AS invalid_user_ids,
       countIf(toDateOrNull(event_date_str) IS NULL) AS invalid_dates
FROM raw_import_staging
```

## Detecting Data Quality Issues

Build a data quality report using OrNull conversions:

```sql
SELECT
    column_name,
    total,
    invalid_count,
    round(invalid_count / total * 100, 2) AS invalid_pct
FROM (
    SELECT
        'user_id'  AS column_name,
        count()    AS total,
        countIf(toInt64OrNull(user_id_raw) IS NULL) AS invalid_count
    FROM raw_data
    UNION ALL
    SELECT
        'revenue',
        count(),
        countIf(toFloat64OrNull(revenue_raw) IS NULL)
    FROM raw_data
)
```

## Summary

ClickHouse's `OrNull` and `OrZero` conversion variants eliminate query failures caused by invalid input by returning NULL or zero respectively. Use `OrNull` when you need to distinguish between valid zero values and parsing failures, and combine with `ifNull()` or `WHERE` filtering to build robust data ingestion and validation pipelines.
