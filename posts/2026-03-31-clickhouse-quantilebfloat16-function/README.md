# How to Use quantileBFloat16() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, quantileBFloat16, Percentile, BFloat16

Description: Learn how quantileBFloat16() uses bfloat16 number compression in ClickHouse to compute fast approximate percentiles with a compact fixed-size data structure.

---

ClickHouse's `quantileBFloat16()` function computes approximate quantiles by compressing input values into the bfloat16 (Brain Float 16) format. Each bfloat16 value uses only 2 bytes and retains the 8-bit exponent of float32, giving it a wide dynamic range with reduced precision. The function stores up to 65,536 unique bfloat16 values and sorts them to find the quantile position, making it very fast while using a predictable fixed amount of memory.

## Basic Syntax

```sql
-- Approximate 95th percentile using bfloat16 compression
SELECT quantileBFloat16(0.95)(response_time) AS p95
FROM requests;
```

For multiple quantiles in one pass, use `quantilesBFloat16()`:

```sql
-- Multiple percentiles from a single pass
SELECT quantilesBFloat16(0.5, 0.75, 0.9, 0.95, 0.99)(latency_ms) AS percentiles
FROM requests;
```

The result is an Array of Float32 values.

## How BFloat16 Compression Works

The bfloat16 format uses:
- 1 sign bit
- 8 exponent bits (same as float32)
- 7 mantissa bits (vs 23 for float32)

```sql
-- Precision is reduced, but the range is preserved
-- Values like 1234.56 may round to 1232 or 1236 in bfloat16
SELECT quantileBFloat16(0.99)(exact_value) AS approx_p99
FROM measurements;
```

Because the exponent is preserved, `quantileBFloat16()` handles values from very small fractions to very large numbers without clamping - unlike `quantileTiming()` which is restricted to 0-30000ms.

## Memory Characteristics

The internal state stores at most 65,536 bfloat16 values (128 KB), regardless of how many rows are processed.

```sql
-- Safe to use on very large tables
SELECT
    count()                             AS total_rows,
    quantileBFloat16(0.99)(event_value) AS p99
FROM billion_row_table;
```

For comparison, `quantileExact()` would require memory proportional to the number of rows, while `quantileBFloat16()` stays bounded at 128 KB.

## Accuracy Characteristics

The relative error introduced by bfloat16 compression is roughly 1% due to the 7-bit mantissa. This means:

```sql
-- For a true value of 1000ms, bfloat16 result may be 990-1010ms
SELECT
    quantile(0.99)(latency_ms)          AS approx_p99,
    quantileBFloat16(0.99)(latency_ms)  AS bfloat16_p99
FROM requests
WHERE event_date = today();
```

For most observability and monitoring workloads, a 1% relative error is well within acceptable bounds.

## Comparing Quantile Functions

```sql
-- Compare accuracy and approach across functions
SELECT
    quantile(0.99)(response_ms)        AS reservoir_p99,
    quantileBFloat16(0.99)(response_ms) AS bfloat16_p99,
    quantileTDigest(0.99)(response_ms)  AS tdigest_p99,
    quantileExact(0.99)(response_ms)    AS exact_p99
FROM http_requests
WHERE event_date = today();
```

`quantileBFloat16()` trades mantissa precision for a larger state capacity (65,536 cells vs t-digest's ~100 centroids) which can improve accuracy in mid-range quantiles.

## Practical Example - Service Latency

```sql
-- Per-service latency using bfloat16
SELECT
    service,
    quantileBFloat16(0.50)(latency_ms) AS p50,
    quantileBFloat16(0.95)(latency_ms) AS p95,
    quantileBFloat16(0.99)(latency_ms) AS p99
FROM spans
WHERE event_date >= today() - 7
GROUP BY service
ORDER BY p99 DESC;
```

## Summary

`quantileBFloat16()` offers a fast, memory-bounded approximate quantile using bfloat16 compression to store up to 65,536 unique values in 128 KB. It handles the full float range (unlike `quantileTiming()`), introduces only around 1% relative error, and performs well on very large datasets. It is a good choice when you need more resolution than t-digest's centroids provide, without the full memory cost of `quantileExact()`.
