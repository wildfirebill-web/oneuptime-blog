# How to Generate Summary Statistics Reports in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Summary Statistics, Percentile, Aggregate Function, Analytics

Description: Learn how to generate summary statistics reports in ClickHouse including mean, median, percentiles, variance, and distribution metrics.

---

## Summary Statistics Overview

Summary statistics describe the distribution of a dataset - central tendency, spread, and shape. ClickHouse provides purpose-built aggregate functions that compute these efficiently over billions of rows.

## Sample Metric Data

```sql
CREATE TABLE response_times
(
    service String,
    recorded_at DateTime,
    latency_ms Float64
)
ENGINE = MergeTree()
ORDER BY (service, recorded_at);
```

## Basic Descriptive Statistics

```sql
SELECT
    service,
    count() AS sample_size,
    min(latency_ms) AS min_ms,
    max(latency_ms) AS max_ms,
    avg(latency_ms) AS mean_ms,
    median(latency_ms) AS median_ms,
    stddevPop(latency_ms) AS stddev_ms,
    varPop(latency_ms) AS variance_ms
FROM response_times
GROUP BY service
ORDER BY mean_ms DESC;
```

## Percentile Distribution

Use `quantiles` to compute multiple percentiles in one pass:

```sql
SELECT
    service,
    quantiles(0.5, 0.75, 0.90, 0.95, 0.99)(latency_ms) AS percentiles
FROM response_times
GROUP BY service;
```

Or use named columns with `quantile`:

```sql
SELECT
    service,
    quantile(0.50)(latency_ms) AS p50,
    quantile(0.90)(latency_ms) AS p90,
    quantile(0.95)(latency_ms) AS p95,
    quantile(0.99)(latency_ms) AS p99
FROM response_times
GROUP BY service
ORDER BY p99 DESC;
```

## Histogram Buckets

Use `histogram` for automatic bucket detection:

```sql
SELECT
    service,
    histogram(10)(latency_ms) AS hist
FROM response_times
GROUP BY service;
```

Or create manual buckets:

```sql
SELECT
    service,
    multiIf(
        latency_ms < 50, '<50ms',
        latency_ms < 100, '50-100ms',
        latency_ms < 200, '100-200ms',
        latency_ms < 500, '200-500ms',
        '500ms+'
    ) AS bucket,
    count() AS count,
    round(count() * 100.0 / sum(count()) OVER (PARTITION BY service), 2) AS pct
FROM response_times
GROUP BY service, bucket
ORDER BY service, bucket;
```

## Coefficient of Variation

Relative spread normalized to the mean:

```sql
SELECT
    service,
    stddevPop(latency_ms) / avg(latency_ms) * 100 AS cv_pct
FROM response_times
GROUP BY service
ORDER BY cv_pct DESC;
```

## Detecting Outliers with IQR

```sql
WITH stats AS (
    SELECT
        service,
        quantile(0.25)(latency_ms) AS q1,
        quantile(0.75)(latency_ms) AS q3
    FROM response_times
    GROUP BY service
)
SELECT r.service, r.latency_ms, r.recorded_at
FROM response_times r
JOIN stats s ON r.service = s.service
WHERE r.latency_ms > s.q3 + 1.5 * (s.q3 - s.q1)
ORDER BY r.latency_ms DESC
LIMIT 100;
```

## Summary

ClickHouse provides `avg`, `median`, `quantile`, `stddevPop`, `varPop`, and `histogram` to cover all major summary statistics needs. Use `quantiles` to compute multiple percentiles in one scan and `histogram` for automatic distribution analysis.
