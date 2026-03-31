# How to Use toInt8(), toInt16(), toInt32(), toInt64() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Type Conversion, Integer, Safe Parsing, Schema

Description: Learn how to use toInt8/16/32/64 and their OrZero/OrNull variants in ClickHouse for signed integer conversion with overflow and error handling.

---

ClickHouse provides `toInt8()`, `toInt16()`, `toInt32()`, and `toInt64()` for converting values to signed integer types. These functions accept strings, floating-point numbers, or other numeric types. For each function, ClickHouse provides two error-handling variants: `toInt32OrZero()` returns 0 on conversion failure, and `toInt32OrNull()` returns NULL on failure. This makes them essential for safely parsing untrusted string data.

## Signed Integer Type Ranges

```text
toInt8   -> Int8   -> -128 to 127
toInt16  -> Int16  -> -32,768 to 32,767
toInt32  -> Int32  -> -2,147,483,648 to 2,147,483,647
toInt64  -> Int64  -> -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807
```

## Basic Usage

```sql
-- Convert string to signed integer
SELECT toInt32('42')   AS from_string;
SELECT toInt32('-999') AS negative_value;
SELECT toInt64('9000000000') AS large_value;

-- Convert float to integer (truncates decimal part)
SELECT toInt32(3.9)  AS truncated;  -- returns 3, not 4
SELECT toInt32(-3.9) AS truncated;  -- returns -3
```

## OrZero Variants - Return 0 on Error

Use `toInt32OrZero()` when invalid input should produce 0 rather than throwing an exception.

```sql
-- Safe conversion: invalid strings return 0
SELECT toInt32OrZero('42')      AS valid_int;    -- returns 42
SELECT toInt32OrZero('abc')     AS invalid_str;  -- returns 0
SELECT toInt32OrZero('')        AS empty_str;    -- returns 0
SELECT toInt32OrZero('3.14')    AS float_str;    -- returns 0 (not a valid int string)

-- Apply to a column with mixed data quality
SELECT
    raw_value,
    toInt32OrZero(raw_value) AS safe_int
FROM raw_input
LIMIT 10;
```

## OrNull Variants - Return NULL on Error

Use `toInt32OrNull()` when you want to distinguish between a valid 0 and a conversion failure.

```sql
-- Invalid strings return NULL, not 0
SELECT toInt32OrNull('42')   AS valid_int;    -- returns 42
SELECT toInt32OrNull('abc')  AS invalid_str;  -- returns NULL
SELECT toInt32OrNull('')     AS empty_str;    -- returns NULL

-- Filter out unparseable values
SELECT
    raw_value,
    toInt32OrNull(raw_value) AS parsed_int
FROM raw_input
WHERE toInt32OrNull(raw_value) IS NOT NULL
LIMIT 10;
```

## Choosing the Right Integer Size

Pick the smallest integer type that fits your data range to minimize storage and improve query performance.

```sql
-- Use toInt8 for small values (e.g., ratings 1-5, age < 127)
SELECT
    user_id,
    toInt8(rating) AS rating_int8
FROM product_ratings
LIMIT 10;

-- Use toInt32 for most general-purpose integers
SELECT
    order_id,
    toInt32(quantity) AS qty
FROM order_lines
LIMIT 10;

-- Use toInt64 for IDs or values that may exceed 2 billion
SELECT
    toInt64(event_count) AS large_count
FROM aggregated_stats
LIMIT 5;
```

## Parsing String Integers from Logs

```sql
-- Parse integer fields from log data
SELECT
    log_line,
    toInt32OrNull(extractAll(log_line, 'status=(\d+)')[1]) AS status_code,
    toInt64OrNull(extractAll(log_line, 'bytes=(\d+)')[1])   AS bytes_transferred
FROM raw_logs
WHERE status_code IS NOT NULL
LIMIT 10;
```

## Type Normalization

When combining data from multiple sources with different numeric types, cast to a consistent type.

```sql
-- Normalize mixed numeric columns to Int64
SELECT
    source_a.record_id,
    toInt64(source_a.value) AS value
FROM source_a

UNION ALL

SELECT
    source_b.record_id,
    toInt64(source_b.value) AS value
FROM source_b
LIMIT 20;
```

## Overflow Handling

When casting a value that is out of range for the target type, ClickHouse throws an error (for the standard variant). Use `accurateCastOrNull()` for safe overflow handling.

```sql
-- toInt8(200) would overflow - use OrNull to handle it safely
SELECT toInt8OrNull(200) AS overflow_result;  -- returns NULL

-- Or use accurateCastOrNull for explicit overflow protection
SELECT accurateCastOrNull(200, 'Int8') AS safe_cast;  -- returns NULL
```

## Summary

`toInt8()`, `toInt16()`, `toInt32()`, and `toInt64()` convert values to signed integers. The `OrZero` variants return 0 on failure; the `OrNull` variants return NULL. Choose `OrNull` when you need to distinguish a valid 0 from a parsing failure, and `OrZero` when 0 is an acceptable default for bad input. Always pick the smallest integer type that covers your data range for optimal storage efficiency.
