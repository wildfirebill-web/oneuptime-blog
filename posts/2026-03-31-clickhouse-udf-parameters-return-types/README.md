# How to Use UDF Parameters and Return Types in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, UDF, Parameter, Return Type, Data Type

Description: Learn how to define and use UDF parameters and return types in ClickHouse, including type casting and handling nullable and composite return values.

---

When defining user-defined functions in ClickHouse, understanding how to specify parameter types and return types is essential for correctness and performance. This post covers type handling for both SQL and executable UDFs.

## SQL UDF Type Inference

SQL UDFs in ClickHouse use lambda expressions and do not require explicit type declarations. ClickHouse infers types from the expression and the caller context:

```sql
CREATE FUNCTION toPercent AS (value, total) ->
    round((value / total) * 100, 2);
```

The return type is inferred as `Float64` based on the division and `round` call.

## Type Casting in SQL UDFs

When you need explicit types, use casting functions inside the lambda:

```sql
CREATE FUNCTION safeDiv AS (numerator, denominator) ->
    if(denominator = 0, toFloat64(0), toFloat64(numerator) / toFloat64(denominator));
```

```sql
SELECT safeDiv(clicks, impressions) AS ctr FROM ad_stats LIMIT 5;
```

## Nullable Parameters

ClickHouse SQL UDFs handle `Nullable` types if the underlying functions accept them:

```sql
CREATE FUNCTION coalesceOrZero AS (val) ->
    ifNull(val, 0);
```

```sql
SELECT coalesceOrZero(nullable_metric) AS safe_metric FROM events;
```

## Executable UDF Parameter Types in XML

For executable UDFs, specify each argument type explicitly:

```text
<function>
    <name>classifyRisk</name>
    <type>executable</type>
    <return_type>String</return_type>
    <argument>
        <type>Float64</type>
        <name>score</name>
    </argument>
    <argument>
        <type>UInt32</type>
        <name>event_count</name>
    </argument>
    <format>TabSeparated</format>
    <command>python3 /var/lib/clickhouse/user_scripts/risk.py</command>
</function>
```

## Supported Return Types for Executable UDFs

Common return types:

```text
String
UInt8, UInt16, UInt32, UInt64
Int8, Int16, Int32, Int64
Float32, Float64
Date, DateTime
```

## Returning Multiple Values

To return multiple values, use a UDF that returns a `Tuple`:

```sql
CREATE FUNCTION splitNameParts AS (full_name) ->
    tuple(
        trim(extract(full_name, '^[^ ]+')),
        trim(extract(full_name, ' (.+)$'))
    );
```

```sql
SELECT
    splitNameParts(full_name).1 AS first_name,
    splitNameParts(full_name).2 AS last_name
FROM users
LIMIT 5;
```

## Type Coercion at Call Site

ClickHouse automatically coerces compatible types when calling UDFs:

```sql
-- UDF defined with Float64 arg, called with Int32 column - ClickHouse casts automatically
SELECT safeDiv(integer_clicks, integer_impressions) AS ctr FROM ad_stats;
```

## Verifying UDF Definitions

```sql
SELECT name, create_query
FROM system.functions
WHERE origin = 'SQLUserDefined'
ORDER BY name;
```

## Common Pitfalls

- Integer division returns integers in SQL UDFs; cast to `Float64` before dividing
- `NULL` propagates through most operations; use `ifNull` or `coalesce` inside UDFs to handle it
- Executable UDF argument order in the XML must match the order values are piped to stdin

## Summary

ClickHouse SQL UDFs rely on type inference, making them easy to write, but you should add explicit casts for numeric operations to avoid integer division pitfalls. Executable UDFs require explicit type declarations in XML. Both support standard ClickHouse types and can return tuples for multi-value results.
