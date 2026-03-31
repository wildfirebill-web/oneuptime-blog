# How to Use Float32 and Float64 Data Types in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Data Types, Float, Float32, Float64

Description: Learn how to use Float32 and Float64 in ClickHouse, including precision trade-offs, NaN and Infinity handling, and when to choose Decimal instead.

---

Floating point types are ideal for scientific measurements, machine learning feature values, ratios, and any data where approximate representation is acceptable. ClickHouse provides two IEEE 754 floating point types - Float32 (single precision) and Float64 (double precision). Understanding their precision limits and special value handling is essential to avoid subtle bugs in analytical queries.

## Float32 vs Float64

Both types follow the IEEE 754 standard, but differ in precision and storage cost.

| Type    | Storage | Significant Digits | Equivalent SQL Type |
|---------|---------|--------------------|---------------------|
| Float32 | 4 bytes | ~7 decimal digits  | FLOAT, REAL         |
| Float64 | 8 bytes | ~15 decimal digits | DOUBLE, DOUBLE PRECISION |

Float32 is suitable when precision beyond 7 digits is not needed and storage efficiency matters. Float64 is the default choice for scientific computations where precision loss would affect results.

## Creating Tables with Float Types

```sql
CREATE TABLE ml_features
(
    sample_id    UInt64,
    feature_1    Float32,
    feature_2    Float32,
    feature_3    Float32,
    score        Float64,   -- Higher precision for final score
    confidence   Float64
)
ENGINE = MergeTree()
ORDER BY sample_id;
```

## Inserting Float Data

```sql
INSERT INTO ml_features
    (sample_id, feature_1, feature_2, feature_3, score, confidence) VALUES
(1, 0.523, -1.204, 3.891, 0.9871234567890123, 0.95),
(2, 1.001, 0.002,  -0.5,  0.4512345678901234, 0.72),
(3, -0.75, 2.333,  1.1,   0.7234567890123456, 0.88);
```

## Precision Limitations

Float32 can lose precision beyond about 7 significant digits. This matters when aggregating many values or when exact decimal representation is required.

```sql
-- Demonstrating Float32 precision limit
SELECT
    toFloat32(0.1) + toFloat32(0.2)  AS float32_sum,
    toFloat64(0.1) + toFloat64(0.2)  AS float64_sum,
    0.1 + 0.2                         AS default_sum;
```

For financial calculations or any domain requiring exact decimal arithmetic, use `Decimal` instead of `Float`.

## NaN and Infinity

ClickHouse Float types support IEEE 754 special values: `nan`, `inf`, and `-inf`. These can appear as results of operations like division by zero or invalid math functions.

```sql
-- Generating special float values
SELECT
    0.0 / 0.0   AS nan_value,
    1.0 / 0.0   AS inf_value,
    -1.0 / 0.0  AS neg_inf_value;
```

### Checking for NaN and Infinity

Use the dedicated predicate functions to detect these values - regular equality checks do not work for NaN.

```sql
SELECT
    sample_id,
    score,
    isNaN(score)     AS is_nan,
    isInfinite(score) AS is_infinite,
    isFinite(score)  AS is_finite
FROM ml_features;
```

### Filtering Out Special Values

```sql
SELECT
    avg(score) AS avg_score,
    avg(confidence) AS avg_confidence
FROM ml_features
WHERE isFinite(score) AND isFinite(confidence);
```

## Aggregation with Float Types

Standard aggregation functions work with float columns. Be aware that summing many Float32 values can accumulate rounding errors.

```sql
SELECT
    count()           AS total_samples,
    avg(feature_1)    AS avg_f1,
    stddevPop(score)  AS score_stddev,
    min(confidence)   AS min_confidence,
    max(confidence)   AS max_confidence
FROM ml_features;
```

For sum-heavy workloads where precision matters, use `sumKahan` which applies Kahan compensated summation to reduce floating point error.

```sql
SELECT sumKahan(score) AS precise_sum
FROM ml_features;
```

## Float vs Decimal: Choosing the Right Type

Use Float when:
- Approximate representation is acceptable (scientific measurements, ML features)
- You need to store very large or very small numbers (1e-300 to 1e+300)
- Storage efficiency and speed outweigh precision

Use Decimal when:
- Exact decimal arithmetic is required (money, accounting, taxes)
- You cannot tolerate rounding errors in sums and comparisons

```sql
-- Float32 rounding error example
SELECT toFloat32(0.1) * 3 AS float_result;
-- May return 0.30000001192092896

-- Decimal exact result
SELECT toDecimal32(0.1, 1) * 3 AS decimal_result;
-- Returns 0.3
```

## Type Casting

```sql
SELECT
    toFloat32(3.14159265358979)  AS as_float32,
    toFloat64(3.14159265358979)  AS as_float64,
    CAST('2.71828', 'Float64')   AS euler_float64;
```

## Summary

Float32 and Float64 are ClickHouse's IEEE 754 floating point types, offering 4 and 8 bytes of storage respectively. Float32 suits memory-sensitive workloads where ~7 digits of precision is enough, while Float64 provides ~15 digits for scientific and high-precision use cases. Always handle NaN and Infinity explicitly using `isNaN` and `isFinite`, and switch to `Decimal` whenever exact decimal arithmetic is required.
