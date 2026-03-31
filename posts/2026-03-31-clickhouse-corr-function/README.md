# How to Use corr() Function for Correlation in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, Correlation, Statistics

Description: Learn how to use corr() in ClickHouse to compute the Pearson correlation coefficient between two variables, with telemetry and observability examples.

---

Correlation tells you not just whether two variables move together (as covariance does) but how strongly and in what direction, on a standardized scale from -1 to 1. ClickHouse's `corr(x, y)` function computes the Pearson correlation coefficient directly as an aggregate, making it straightforward to answer questions like "does CPU usage predict latency?" or "are error rate and throughput inversely related?" across billions of rows of telemetry data.

## Understanding the Pearson Correlation Coefficient

`corr(x, y)` returns a Float64 in the range [-1, 1]:

- **1.0** - perfect positive linear relationship (x rises, y rises proportionally)
- **-1.0** - perfect negative linear relationship (x rises, y falls proportionally)
- **0.0** - no linear relationship
- Values between 0.5 and 1.0 (or -1.0 and -0.5) indicate strong correlation.

Mathematically it is equivalent to `covarSamp(x, y) / (stddevSamp(x) * stddevSamp(y))`.

```sql
-- Basic syntax
SELECT corr(x_column, y_column) FROM table_name;
```

## Basic Usage

### CPU vs Latency Correlation

```sql
-- Does CPU usage predict API latency?
SELECT
    service_name,
    round(corr(cpu_percent, response_time_ms), 4) AS cpu_latency_corr,
    count() AS samples
FROM host_metrics
JOIN request_logs USING (host_name)
WHERE host_metrics.metric_time >= now() - INTERVAL 1 HOUR
GROUP BY service_name
ORDER BY abs(cpu_latency_corr) DESC;
```

### Error Rate vs Latency Correlation

```sql
-- Compute per-minute aggregates then correlate
SELECT
    round(corr(avg_latency, error_rate), 4) AS latency_error_corr
FROM (
    SELECT
        toStartOfMinute(timestamp)             AS minute,
        avg(response_time_ms)                  AS avg_latency,
        countIf(status_code >= 500) / count()  AS error_rate
    FROM request_logs
    WHERE log_date = today()
    GROUP BY minute
);
```

## Correlation Matrix Across Multiple Metrics

Compute all pairwise correlations in a single scan by repeating `corr()` for each pair.

```sql
SELECT
    round(corr(latency_ms, cpu_pct),  4) AS r_latency_cpu,
    round(corr(latency_ms, mem_pct),  4) AS r_latency_mem,
    round(corr(latency_ms, rps),      4) AS r_latency_rps,
    round(corr(cpu_pct,    mem_pct),  4) AS r_cpu_mem,
    round(corr(cpu_pct,    rps),      4) AS r_cpu_rps,
    round(corr(mem_pct,    rps),      4) AS r_mem_rps
FROM (
    SELECT
        toStartOfMinute(timestamp)  AS minute,
        avg(response_time_ms)       AS latency_ms,
        avg(cpu_percent)            AS cpu_pct,
        avg(memory_percent)         AS mem_pct,
        count()                     AS rps
    FROM request_logs
    JOIN host_metrics USING (host_name)
    WHERE log_date = today()
    GROUP BY minute
);
```

## Temporal Correlation Analysis

Track how correlation between metrics evolves over time.

```sql
-- Hourly CPU-latency correlation trend over the past week
SELECT
    toStartOfHour(timestamp) AS hour,
    round(corr(cpu_percent, response_time_ms), 4) AS cpu_latency_corr,
    count() AS samples
FROM host_metrics
JOIN request_logs USING (host_name)
WHERE host_metrics.metric_time >= now() - INTERVAL 7 DAY
GROUP BY hour
ORDER BY hour;
```

When the correlation jumps from 0.2 to 0.9 on a specific hour, that is a strong signal of a resource-constrained incident.

## Interpreting Correlation Strength

```sql
-- Classify correlation strength per endpoint
SELECT
    endpoint,
    round(corr(cpu_percent, response_time_ms), 4) AS r,
    CASE
        WHEN abs(corr(cpu_percent, response_time_ms)) >= 0.9 THEN 'Very Strong'
        WHEN abs(corr(cpu_percent, response_time_ms)) >= 0.7 THEN 'Strong'
        WHEN abs(corr(cpu_percent, response_time_ms)) >= 0.5 THEN 'Moderate'
        WHEN abs(corr(cpu_percent, response_time_ms)) >= 0.3 THEN 'Weak'
        ELSE 'Negligible'
    END AS strength
FROM request_logs
JOIN host_metrics USING (host_name)
WHERE log_date = today()
GROUP BY endpoint
ORDER BY abs(r) DESC;
```

## Correlation in Materialized Views

```sql
CREATE TABLE daily_corr_stats
(
    stat_date Date,
    service   String,
    corr_state AggregateFunction(corr, Float64, Float64)
)
ENGINE = AggregatingMergeTree()
ORDER BY (stat_date, service);

CREATE MATERIALIZED VIEW mv_daily_corr
TO daily_corr_stats
AS
SELECT
    toDate(timestamp)                                 AS stat_date,
    service_name                                      AS service,
    corrState(toFloat64(cpu_percent),
              toFloat64(response_time_ms))            AS corr_state
FROM request_logs
JOIN host_metrics USING (host_name)
GROUP BY stat_date, service;

-- Query merged correlation
SELECT
    stat_date,
    service,
    round(corrMerge(corr_state), 4) AS cpu_latency_corr
FROM daily_corr_stats
GROUP BY stat_date, service
ORDER BY stat_date DESC;
```

## Caveats

Pearson correlation measures only linear relationships. If two metrics have a non-linear relationship (e.g., U-shaped), `corr()` may return a near-zero result even though they are strongly related. Additionally, correlation does not imply causation - always interpret results in context.

```sql
-- Verify the relationship is reasonably linear by also looking at quartile values
SELECT
    quantile(0.25)(response_time_ms) AS p25_latency,
    quantile(0.50)(response_time_ms) AS p50_latency,
    quantile(0.75)(response_time_ms) AS p75_latency,
    quantile(0.95)(response_time_ms) AS p95_latency,
    corr(cpu_percent, response_time_ms) AS cpu_latency_corr
FROM request_logs
WHERE log_date = today();
```

## Summary

`corr(x, y)` computes the Pearson correlation coefficient between two numeric columns, returning a value in [-1, 1] that measures linear co-movement in a scale-free way. It is the go-to function for telemetry correlation analysis - identifying whether CPU, memory, throughput, or other system signals are predictive of latency or error rates. Combine it with `AggregatingMergeTree` materialized views for incremental daily correlation tracking across services and infrastructure components.
