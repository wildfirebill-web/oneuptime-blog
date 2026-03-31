# How to Run Statistical Analysis on ClickHouse Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Statistical Analysis, Aggregate Function, Percentile, Correlation, Analytics

Description: Learn how to run descriptive statistics, percentiles, correlations, and hypothesis tests on ClickHouse data using built-in aggregate functions.

---

ClickHouse includes a rich set of statistical aggregate functions that go far beyond simple COUNT and AVG. This post covers practical statistical analysis patterns - from basic descriptive statistics to correlation and hypothesis testing.

## Descriptive Statistics

Get a full statistical summary in a single query:

```sql
SELECT
    count()                     AS n,
    avg(response_time_ms)       AS mean,
    median(response_time_ms)    AS median,
    stddevSamp(response_time_ms) AS stddev,
    varSamp(response_time_ms)   AS variance,
    min(response_time_ms)       AS min,
    max(response_time_ms)       AS max,
    quantile(0.25)(response_time_ms) AS p25,
    quantile(0.75)(response_time_ms) AS p75,
    quantile(0.95)(response_time_ms) AS p95,
    quantile(0.99)(response_time_ms) AS p99
FROM http_logs
WHERE timestamp >= now() - INTERVAL 1 DAY;
```

## Percentiles and Quantiles

ClickHouse provides multiple quantile estimators with different accuracy-speed tradeoffs:

```sql
-- Exact (slow on large datasets)
SELECT quantileExact(0.95)(response_time_ms) FROM http_logs;

-- Approximate using t-digest (fast, accurate)
SELECT quantileTDigest(0.95)(response_time_ms) FROM http_logs;

-- Multiple quantiles at once
SELECT quantiles(0.5, 0.9, 0.95, 0.99)(response_time_ms) FROM http_logs;
```

## Frequency Distribution (Histogram)

```sql
SELECT
    histogram(20)(response_time_ms) AS hist
FROM http_logs
WHERE timestamp >= now() - INTERVAL 7 DAY;
```

For bucket-based histograms:

```sql
SELECT
    floor(response_time_ms / 100) * 100 AS bucket,
    count() AS cnt
FROM http_logs
GROUP BY bucket
ORDER BY bucket;
```

## Correlation Analysis

Measure the relationship between two numeric columns:

```sql
SELECT
    corr(response_time_ms, payload_bytes) AS correlation,
    covarSamp(response_time_ms, payload_bytes) AS covariance
FROM http_logs;
```

A value near 1 or -1 indicates a strong linear relationship; near 0 indicates no linear relationship.

## Skewness and Kurtosis

```sql
SELECT
    skewSamp(response_time_ms)  AS skewness,
    kurtSamp(response_time_ms)  AS kurtosis
FROM http_logs;
```

Positive skewness indicates a long right tail - common in latency distributions.

## Student T-Test (A/B Testing)

Compare means between two groups:

```sql
SELECT
    studentTTest(response_time_ms, toUInt8(endpoint = '/api/v2/search'))
    AS ttest_result
FROM http_logs
WHERE endpoint IN ('/api/v1/search', '/api/v2/search');
```

The result is a tuple `(t-statistic, p-value)`. A p-value below 0.05 suggests a statistically significant difference.

## Welch T-Test

When variances between groups are unequal:

```sql
SELECT
    welchTTest(response_time_ms, toUInt8(endpoint = '/api/v2/search'))
    AS welch_result
FROM http_logs
WHERE endpoint IN ('/api/v1/search', '/api/v2/search');
```

## Moving Averages and Rolling Statistics

```sql
SELECT
    toStartOfHour(timestamp) AS hour,
    avg(response_time_ms) AS avg_rt,
    avg(avg(response_time_ms)) OVER (
        ORDER BY hour
        ROWS BETWEEN 5 PRECEDING AND CURRENT ROW
    ) AS rolling_avg_6h
FROM http_logs
GROUP BY hour
ORDER BY hour;
```

## Summary

ClickHouse provides extensive built-in statistical functions including descriptive statistics, multiple quantile estimators, correlation, skewness, kurtosis, and parametric hypothesis tests (Student T, Welch T). These functions run at ClickHouse speeds across billions of rows, making statistical analysis on large datasets practical without exporting data to Python or R.
