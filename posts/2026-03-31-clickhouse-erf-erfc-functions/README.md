# How to Use erf() and erfc() Error Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Math Function, Statistics, Probability

Description: Learn how erf() and erfc() compute the error function and its complement in ClickHouse, with examples for z-score probabilities and normal distribution calculations.

---

The error function `erf(x)` and its complement `erfc(x)` are special mathematical functions closely related to the normal distribution. They are used to compute the probability that a normally distributed random variable falls within a given range. While not as commonly used in SQL as simpler math functions, they are essential for statistical quality control, anomaly detection thresholds, and A/B test significance calculations directly in ClickHouse queries without requiring external computation.

## Function Signatures

```text
erf(x)    -- error function; erf(x) = (2/sqrt(pi)) * integral from 0 to x of exp(-t^2) dt
erfc(x)   -- complementary error function; erfc(x) = 1 - erf(x)
```

Both return Float64. `erf()` ranges from -1 to 1. `erfc()` ranges from 0 to 2. For large positive x, `erf(x)` approaches 1 and `erfc(x)` approaches 0.

## Basic Values

```sql
SELECT
    round(erf(0),   4)  AS erf_0,
    round(erf(1),   4)  AS erf_1,
    round(erf(2),   4)  AS erf_2,
    round(erf(-1),  4)  AS erf_neg1,
    round(erfc(0),  4)  AS erfc_0,
    round(erfc(1),  4)  AS erfc_1,
    round(erf(1) + erfc(1), 4) AS sum_is_1;
```

`erf(0)` = 0, `erf(1)` ≈ 0.8427, `erf(2)` ≈ 0.9953. `erfc(0)` = 1.

## Relationship to the Normal Distribution CDF

The CDF of the standard normal distribution (Phi function) relates to `erf()` as:

```text
Phi(x) = 0.5 * (1 + erf(x / sqrt(2)))
```

This allows computing the probability that a standard normal variable falls below x.

```sql
SELECT
    z_score,
    round(0.5 * (1 + erf(z_score / sqrt(2))), 4) AS normal_cdf,
    round(1 - 0.5 * (1 + erf(z_score / sqrt(2))), 4) AS upper_tail_prob
FROM (
    SELECT arrayJoin([-3.0, -2.0, -1.0, 0.0, 1.0, 1.645, 1.96, 2.0, 2.576, 3.0]) AS z_score
)
ORDER BY z_score;
```

This produces the familiar standard normal table. `z=1.96` gives CDF ≈ 0.975, confirming the 95% two-tailed confidence interval threshold.

## Computing Probability Within a Standard Deviation Range

The probability of a standard normal variable falling between -z and +z is `erf(z / sqrt(2))`.

```sql
SELECT
    z,
    round(erf(z / sqrt(2)), 4) AS prob_within_z_sigmas,
    round((1 - erf(z / sqrt(2))) / 2, 4) AS tail_prob_each_side
FROM (
    SELECT arrayJoin([1.0, 2.0, 3.0]) AS z
);
```

This confirms: 68.27% within 1 sigma, 95.45% within 2 sigma, 99.73% within 3 sigma.

## Setting Up Anomaly Detection Data

Create a table of metric readings to demonstrate using `erf()` for z-score based anomaly scoring.

```sql
CREATE TABLE metric_readings
(
    metric_id  UInt64,
    ts         DateTime,
    value      Float64
)
ENGINE = MergeTree
ORDER BY (metric_id, ts);

INSERT INTO metric_readings VALUES
(1, '2024-12-01 00:00:00', 100.2),
(1, '2024-12-01 01:00:00', 99.8),
(1, '2024-12-01 02:00:00', 101.1),
(1, '2024-12-01 03:00:00', 98.5),
(1, '2024-12-01 04:00:00', 145.0),
(1, '2024-12-01 05:00:00', 100.5);
```

## Anomaly Probability Score Using erf()

Compute a per-row anomaly probability by converting z-scores to tail probabilities using `erfc()`.

```sql
WITH stats AS (
    SELECT
        metric_id,
        avg(value) AS mean_val,
        stddevPop(value) AS std_val
    FROM metric_readings
    GROUP BY metric_id
)
SELECT
    r.metric_id,
    r.ts,
    r.value,
    round((r.value - s.mean_val) / nullIf(s.std_val, 0), 2) AS z_score,
    round(erfc(abs(r.value - s.mean_val) / (s.std_val * sqrt(2))), 6) AS anomaly_p_value,
    abs(r.value - s.mean_val) / nullIf(s.std_val, 0) > 3 AS is_3sigma_outlier
FROM metric_readings r
JOIN stats s ON r.metric_id = s.metric_id
ORDER BY anomaly_p_value;
```

## Summary

`erf(x)` and `erfc(x)` are the building blocks for normal distribution probability computations in ClickHouse. Use them to compute normal CDF values from z-scores, find the probability of a value falling within a given number of standard deviations, and score anomalies based on their tail probability. The key formula to remember is: `normal_cdf(z) = 0.5 * (1 + erf(z / sqrt(2)))`. `erfc()` is numerically preferable when computing small upper-tail probabilities because it avoids catastrophic cancellation in `1 - erf(x)` for large x.
