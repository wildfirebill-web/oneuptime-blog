# How to Build Box Plot Statistics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Statistics, Box Plot, Percentile, Analytics

Description: Learn how to compute box plot statistics in ClickHouse using quantile functions to visualize data spread and detect outliers efficiently.

---

## What Is a Box Plot?

A box plot (also called a box-and-whisker plot) summarizes a distribution using five numbers: minimum, first quartile (Q1), median (Q2), third quartile (Q3), and maximum. It is useful for spotting skewed distributions, outliers, and comparing groups.

## Computing the Five-Number Summary

ClickHouse's `quantile` and `quantiles` functions make this straightforward:

```sql
SELECT
    min(latency_ms)                     AS min_val,
    quantile(0.25)(latency_ms)          AS q1,
    quantile(0.50)(latency_ms)          AS median,
    quantile(0.75)(latency_ms)          AS q3,
    max(latency_ms)                     AS max_val
FROM request_logs
WHERE toDate(created_at) = today();
```

Using `quantiles` in a single pass is more efficient:

```sql
SELECT
    quantiles(0.25, 0.50, 0.75)(latency_ms) AS quartiles
FROM request_logs
WHERE toDate(created_at) = today();
```

This returns an array: `[Q1, median, Q3]`.

## Computing the IQR and Whiskers

The interquartile range (IQR) is `Q3 - Q1`. Standard whiskers extend to 1.5 * IQR beyond the quartiles:

```sql
WITH
    quantile(0.25)(latency_ms) AS q1,
    quantile(0.75)(latency_ms) AS q3,
    (q3 - q1) AS iqr
SELECT
    q1,
    q3,
    iqr,
    q1 - 1.5 * iqr AS lower_fence,
    q3 + 1.5 * iqr AS upper_fence
FROM request_logs
WHERE toDate(created_at) = today();
```

## Identifying Outliers

Values outside the whiskers are outliers. You can count them:

```sql
WITH stats AS (
    SELECT
        quantile(0.25)(latency_ms) AS q1,
        quantile(0.75)(latency_ms) AS q3
    FROM request_logs
    WHERE toDate(created_at) = today()
)
SELECT count() AS outlier_count
FROM request_logs, stats
WHERE toDate(created_at) = today()
  AND (
    latency_ms < q1 - 1.5 * (q3 - q1)
    OR latency_ms > q3 + 1.5 * (q3 - q1)
  );
```

## Box Plot Stats Per Group

Break down statistics by a categorical dimension to compare services:

```sql
SELECT
    service_name,
    min(latency_ms)            AS min_val,
    quantile(0.25)(latency_ms) AS q1,
    quantile(0.50)(latency_ms) AS median,
    quantile(0.75)(latency_ms) AS q3,
    max(latency_ms)            AS max_val
FROM service_metrics
WHERE toDate(ts) >= today() - 7
GROUP BY service_name
ORDER BY median DESC;
```

## Using quantileTDigest for Large Datasets

For very large tables, `quantileTDigest` offers faster approximate results:

```sql
SELECT
    quantileTDigest(0.25)(latency_ms) AS q1,
    quantileTDigest(0.50)(latency_ms) AS median,
    quantileTDigest(0.75)(latency_ms) AS q3
FROM request_logs;
```

## Summary

ClickHouse's `quantile`, `quantiles`, and `quantileTDigest` functions give you everything needed to build box plot statistics directly in SQL. Pair them with the IQR formula to detect outliers, and group by dimensions to compare distributions across services or time windows.
