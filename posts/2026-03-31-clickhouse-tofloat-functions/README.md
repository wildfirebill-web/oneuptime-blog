# How to Use toFloat32() and toFloat64() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Type Conversion, Float, Decimal Parsing, Numeric

Description: Learn how to use toFloat32() and toFloat64() in ClickHouse to convert strings and integers to floating-point types with OrZero and OrNull error handling.

---

`toFloat32()` and `toFloat64()` convert values to IEEE 754 floating-point types. `Float32` uses 4 bytes and provides about 7 decimal digits of precision. `Float64` uses 8 bytes and provides about 15 decimal digits. Both have `OrZero` and `OrNull` variants for safe conversion. For financial calculations requiring exact precision, use `Decimal` types instead.

## Basic Usage

```sql
-- Convert string to Float64
SELECT toFloat64('3.14159') AS pi;
SELECT toFloat64('1.23e5')  AS scientific;  -- scientific notation

-- Convert integer to Float64
SELECT toFloat64(42) AS int_to_float;

-- toFloat32 for lower-precision use cases
SELECT toFloat32('3.14') AS float32_val;
```

## Float32 vs Float64 Precision

```sql
-- Float32 has about 7 decimal digits of precision
SELECT
    toFloat32(1.23456789) AS float32_precision,  -- some digits lost
    toFloat64(1.23456789) AS float64_precision;  -- more digits preserved
```

## OrZero and OrNull Variants

```sql
-- OrZero: returns 0.0 on invalid input
SELECT toFloat64OrZero('not a number') AS invalid;  -- returns 0
SELECT toFloat64OrZero('3.14')         AS valid;    -- returns 3.14
SELECT toFloat64OrZero('')             AS empty;    -- returns 0

-- OrNull: returns NULL on invalid input
SELECT toFloat64OrNull('not a number') AS invalid;  -- returns NULL
SELECT toFloat64OrNull('3.14')         AS valid;    -- returns 3.14

-- Apply to column data safely
SELECT
    price_str,
    toFloat64OrNull(price_str) AS price_float
FROM product_import
WHERE toFloat64OrNull(price_str) IS NOT NULL
LIMIT 10;
```

## Parsing Decimal Strings from External Data

When importing data from CSV files or APIs, numeric values often arrive as strings. Use `toFloat64OrNull` for safe parsing.

```sql
-- Parse price and weight from a product import
SELECT
    product_id,
    product_name,
    toFloat64OrNull(raw_price)  AS price,
    toFloat64OrNull(raw_weight) AS weight_kg
FROM product_import_staging
LIMIT 10;

-- Validate: flag rows where conversion failed
SELECT
    product_id,
    raw_price,
    toFloat64OrNull(raw_price) IS NULL AS price_parse_failed
FROM product_import_staging
WHERE toFloat64OrNull(raw_price) IS NULL
LIMIT 10;
```

## Converting Integers for Mathematical Operations

Some mathematical functions require float input. Use `toFloat64()` to prepare integer columns.

```sql
-- Compute percentage as float
SELECT
    category,
    count()                                            AS item_count,
    toFloat64(count()) / toFloat64(sum(count()) OVER ()) * 100 AS pct_of_total
FROM products
GROUP BY category
ORDER BY item_count DESC
LIMIT 10;

-- Square root requires float input
SELECT
    user_id,
    sqrt(toFloat64(score)) AS score_sqrt
FROM user_scores
LIMIT 10;
```

## Float Arithmetic Precision Warning

Floating-point arithmetic is subject to rounding errors. For financial calculations, use `Decimal` types.

```sql
-- Floating-point precision issue
SELECT
    toFloat64(0.1) + toFloat64(0.2) AS float_result,  -- may not be exactly 0.3
    toDecimal64(0.1, 10) + toDecimal64(0.2, 10) AS decimal_result;  -- exact
```

## Handling Infinity and NaN

`Float32` and `Float64` can hold special values like infinity and NaN. Check for them explicitly.

```sql
-- inf and nan from division by zero (Float division, not integer)
SELECT
    isNaN(0.0 / 0.0)    AS is_nan,
    isInfinite(1.0 / 0.0) AS is_inf;

-- Clean up NaN and Inf values
SELECT
    value,
    if(isNaN(value) OR isInfinite(value), 0.0, value) AS clean_value
FROM float_data
LIMIT 10;
```

## Storage and Performance Considerations

```sql
-- Float32 uses half the storage of Float64
-- Use Float32 for large tables where a few digits of precision are sufficient
CREATE TABLE measurements
(
    sensor_id     UInt32,
    measured_at   DateTime,
    temperature   Float32,   -- 4 bytes, sufficient for temperature (7 sig. digits)
    precise_value Float64    -- 8 bytes, for values needing 15 sig. digits
)
ENGINE = MergeTree()
ORDER BY (sensor_id, measured_at);
```

## Summary

`toFloat32()` and `toFloat64()` convert values to floating-point types. Use `Float64` for general-purpose decimal arithmetic and `Float32` only when precision above 7 significant digits is not required and storage savings matter. Always use the `OrNull` or `OrZero` variants when parsing untrusted string data. For financial or exact decimal arithmetic, prefer `Decimal32`, `Decimal64`, or `Decimal128` instead of floating-point types.
