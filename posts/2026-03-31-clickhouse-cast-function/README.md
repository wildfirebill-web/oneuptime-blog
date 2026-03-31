# How to Use CAST() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Type Conversion, CAST, SQL, Schema Alignment

Description: Learn how to use CAST() in ClickHouse for standard SQL type casting, schema alignment, and complex type conversions across all supported types.

---

`CAST(x AS type)` and its equivalent `CAST(x, 'type')` are the standard SQL way to convert values between types in ClickHouse. Unlike the `toInt32()`, `toFloat64()`, and other dedicated conversion functions, `CAST` follows the SQL standard syntax and accepts type names as strings. It supports all ClickHouse types including nullable, array, tuple, map, and fixed string types.

## Basic Syntax

```text
CAST(x AS TypeName)
CAST(x, 'TypeName')   -- alternative ClickHouse-specific syntax
```

Both forms are equivalent. The SQL standard form (`AS TypeName`) is preferred for portability.

## Basic Type Conversions

```sql
-- Cast string to integer
SELECT CAST('42' AS Int32) AS int_value;

-- Cast string to float
SELECT CAST('3.14' AS Float64) AS float_value;

-- Cast integer to string
SELECT CAST(42 AS String) AS str_value;

-- Cast date string to Date
SELECT CAST('2025-01-15' AS Date) AS date_value;

-- Cast to DateTime
SELECT CAST('2025-01-15 14:30:00' AS DateTime) AS dt_value;
```

## Schema Alignment in UNION ALL

One of the most common uses of `CAST` is aligning column types between branches of a `UNION ALL`.

```sql
-- Align types from different tables with different schemas
SELECT
    order_id,
    CAST(amount AS Float64)    AS amount,
    CAST(created_at AS String) AS created_at_str
FROM legacy_orders

UNION ALL

SELECT
    order_id,
    amount,
    toString(created_at) AS created_at_str
FROM modern_orders

ORDER BY created_at_str DESC
LIMIT 20;
```

## CAST to Nullable Types

```sql
-- Cast to Nullable type (allows NULL in the result)
SELECT CAST(42 AS Nullable(Int32)) AS nullable_int;
SELECT CAST(NULL AS Nullable(String)) AS nullable_str;

-- Cast a column to Nullable for use in expressions
SELECT
    user_id,
    CAST(age AS Nullable(Int32)) AS age_nullable
FROM users
LIMIT 5;
```

## CAST to FixedString

`FixedString(n)` stores exactly `n` bytes, padding with null bytes if shorter. Use it for fixed-width values like hash digests, codes, and identifiers.

```sql
-- Cast a hash string to FixedString
SELECT CAST('abc123' AS FixedString(6)) AS fixed_str;

-- Cast MD5 hex output to FixedString(32)
SELECT CAST(hex(MD5('hello world')) AS FixedString(32)) AS fixed_md5;
```

## CAST to Decimal

```sql
-- Cast float to Decimal with specified scale
SELECT CAST(3.14159 AS Decimal64(4)) AS decimal_val;
-- Returns: 3.1416 (rounded to 4 decimal places)

-- Cast string price to Decimal
SELECT CAST('99.99' AS Decimal64(2)) AS price;
```

## CAST to Array Types

```sql
-- Cast a string representation of an array to an actual Array type
SELECT CAST('[1, 2, 3]' AS Array(Int32)) AS int_array;

-- Cast array elements to a different type
SELECT CAST([1, 2, 3] AS Array(Float64)) AS float_array;
```

## CAST vs Dedicated Conversion Functions

```sql
-- These are equivalent
SELECT
    CAST('42' AS Int32)    AS cast_version,
    toInt32('42')          AS toInt32_version;

-- CAST requires a string type name; toInt32() is shorter for simple cases
-- CAST is preferred when writing portable SQL or complex type names
SELECT
    CAST(value AS Nullable(Decimal64(4))) AS cast_nullable_decimal
FROM my_table
LIMIT 5;
```

The dedicated functions (like `toInt32`) are shorter for simple conversions. Use `CAST` when:
- Writing SQL that should be portable to other databases
- Casting to complex or nested types that have no dedicated function
- Casting to `Nullable(T)` or `FixedString(n)` types

## Arithmetic After CAST

```sql
-- Cast before arithmetic to control precision
SELECT
    product_id,
    CAST(price_int AS Float64) / 100.0 AS price_float,
    CAST(quantity AS Int64) * CAST(price_int AS Int64) AS total_cents
FROM products
LIMIT 10;
```

## Type Checking After CAST

```sql
-- Verify the output type
SELECT
    toTypeName(CAST(42 AS Int8))              AS int8_type,
    toTypeName(CAST(42 AS Float64))           AS float64_type,
    toTypeName(CAST('x' AS FixedString(4)))   AS fixed_str_type,
    toTypeName(CAST(1 AS Nullable(UInt32)))   AS nullable_uint_type;
```

## Summary

`CAST(x AS type)` is the standard SQL type conversion operator in ClickHouse. It supports all ClickHouse types including `Nullable(T)`, `FixedString(n)`, `Array(T)`, `Decimal(p, s)`, and more. Use it for schema alignment in `UNION ALL`, casting to complex types, and when you prefer standard SQL syntax over dedicated functions like `toInt32()`. For simple numeric conversions, the dedicated `toXxx()` functions are more concise.
