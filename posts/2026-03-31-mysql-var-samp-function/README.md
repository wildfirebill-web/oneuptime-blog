# How to Use VAR_SAMP() Function in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Aggregate Function, Statistics

Description: Learn how to use MySQL's VAR_SAMP() aggregate function to calculate sample variance, with examples comparing it to VAR_POP() and STDDEV_SAMP().

---

## What Is VAR_SAMP()?

`VAR_SAMP()` computes the sample variance of a set of numeric values. Sample variance uses Bessel's correction - it divides by `N-1` rather than `N` - making it the statistically correct measure when your data is a sample drawn from a larger population.

Formula: `VAR_SAMP = SUM((x - mean)^2) / (N - 1)`

MySQL also provides `VAR_POP()` (population variance, divides by `N`) and `VARIANCE()` (an alias for `VAR_POP()`).

## When to Use VAR_SAMP() vs VAR_POP()

| Function | Divides by | Use when |
|---|---|---|
| `VAR_SAMP()` | N - 1 | Data is a sample from a larger population |
| `VAR_POP()` | N | Data is the entire population |
| `VARIANCE()` | N | Alias for VAR_POP() |

In most analytical applications, `VAR_SAMP()` is the correct choice because you are typically working with a sample.

## Basic Usage

```sql
CREATE TABLE measurements (
  id INT AUTO_INCREMENT PRIMARY KEY,
  sensor_id INT,
  reading DECIMAL(10,4)
);

INSERT INTO measurements (sensor_id, reading) VALUES
  (1, 23.5), (1, 24.1), (1, 22.8), (1, 25.0), (1, 23.9),
  (2, 100.2), (2, 99.8), (2, 101.0), (2, 100.5);

SELECT
  sensor_id,
  COUNT(*) AS n,
  AVG(reading) AS mean,
  VAR_SAMP(reading) AS sample_variance,
  VAR_POP(reading) AS population_variance
FROM measurements
GROUP BY sensor_id;
```

## VAR_SAMP() Returns NULL for Single Values

With only one row, sample variance is undefined (division by zero):

```sql
SELECT VAR_SAMP(reading) FROM measurements WHERE id = 1;
-- Result: NULL
```

`VAR_POP()` returns 0 for a single value (variance of one point from itself is zero).

## Relationship to STDDEV_SAMP()

Standard deviation is the square root of variance:

```sql
SELECT
  sensor_id,
  VAR_SAMP(reading) AS sample_variance,
  STDDEV_SAMP(reading) AS sample_std_dev,
  SQRT(VAR_SAMP(reading)) AS manual_std_dev  -- same as STDDEV_SAMP
FROM measurements
GROUP BY sensor_id;
```

## Practical Example: Detecting Anomalous Sensors

Find sensors with unusually high variance in readings (potentially faulty):

```sql
SELECT
  sensor_id,
  ROUND(AVG(reading), 4) AS mean_reading,
  ROUND(VAR_SAMP(reading), 6) AS variance,
  ROUND(STDDEV_SAMP(reading), 4) AS std_dev,
  COUNT(*) AS sample_count
FROM measurements
GROUP BY sensor_id
HAVING VAR_SAMP(reading) > 1.0
ORDER BY variance DESC;
```

## Using VAR_SAMP() with OVER() as a Window Function

In MySQL 8.0+, `VAR_SAMP()` can be used as a window function:

```sql
SELECT
  id,
  sensor_id,
  reading,
  VAR_SAMP(reading) OVER (
    PARTITION BY sensor_id
    ORDER BY id
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_variance
FROM measurements;
```

This computes the running sample variance as new readings are added.

## NULL Handling

`VAR_SAMP()` ignores `NULL` values:

```sql
SELECT VAR_SAMP(val) FROM (
  SELECT 10 AS val UNION ALL SELECT 20 UNION ALL SELECT NULL UNION ALL SELECT 30
) t;
-- Computes variance of [10, 20, 30] only
```

## Summary

`VAR_SAMP()` calculates sample variance using Bessel's correction (N-1 denominator), making it the statistically appropriate choice when your dataset is a sample rather than a complete population. Use `STDDEV_SAMP()` for the more interpretable standard deviation, and `VAR_POP()` only when analyzing a complete population.
