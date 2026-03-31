# How to Use varSamp() and varPop() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Aggregate Function, Variance, Statistics

Description: Learn how to use varSamp() and varPop() in ClickHouse to compute sample and population variance, including when to choose each and practical SQL examples.

---

Variance measures how spread out a set of values is around their mean. ClickHouse provides two variance functions: `varSamp()` for sample variance and `varPop()` for population variance. Choosing the right one depends on whether your dataset represents a sample drawn from a larger population or the complete population itself. Both functions are aggregate functions that work seamlessly inside GROUP BY queries, materialized views, and window functions.

## varSamp() vs varPop() - The Key Difference

The difference lies in the denominator used in the variance formula:

- `varPop(x)` divides by N (the total number of values) - use this when your data IS the entire population.
- `varSamp(x)` divides by N-1 (Bessel's correction) - use this when your data is a SAMPLE from a larger population.

In most analytical and monitoring workloads you are working with a sample (a time window, a subset of users), so `varSamp()` is typically more appropriate.

```sql
-- Population variance
SELECT varPop(response_time_ms) FROM request_logs;

-- Sample variance
SELECT varSamp(response_time_ms) FROM request_logs;
```

## Basic Usage Examples

### Response Time Variability

```sql
-- Measure spread of API response times per endpoint
SELECT
    endpoint,
    avg(response_time_ms)    AS mean_latency,
    varSamp(response_time_ms) AS latency_variance,
    count()                   AS request_count
FROM request_logs
WHERE log_date = today()
GROUP BY endpoint
ORDER BY latency_variance DESC
LIMIT 20;
```

High variance in latency indicates inconsistent performance, even if the mean looks acceptable.

### Error Rate Variance Across Services

```sql
-- Daily variance of error rates per service
SELECT
    service_name,
    toDate(timestamp) AS log_date,
    varSamp(CASE WHEN status_code >= 500 THEN 1.0 ELSE 0.0 END) AS error_rate_variance
FROM request_logs
GROUP BY service_name, log_date
ORDER BY log_date DESC, error_rate_variance DESC;
```

## Population Variance with varPop()

Use `varPop()` when your table contains the complete set of observations with no sampling involved, such as a fixed test run or a complete census.

```sql
-- Population variance of scores in a completed exam
SELECT
    exam_id,
    varPop(score)  AS score_variance_pop,
    varSamp(score) AS score_variance_samp,
    count()        AS student_count
FROM exam_results
GROUP BY exam_id;
```

For large N the two values converge. For small N (under 30), the difference is meaningful.

## Using varSamp() for Anomaly Detection

Variance is the foundation for z-score based anomaly detection. You can compute it in a subquery and then flag outliers.

```sql
-- Flag requests whose latency deviates more than 2 standard deviations from the mean
-- sqrt(varSamp()) gives the standard deviation
SELECT
    request_id,
    endpoint,
    response_time_ms,
    avg_ms,
    sqrt_var,
    (response_time_ms - avg_ms) / sqrt_var AS z_score
FROM (
    SELECT
        request_id,
        endpoint,
        response_time_ms,
        avg(response_time_ms) OVER (PARTITION BY endpoint) AS avg_ms,
        sqrt(varSamp(response_time_ms) OVER (PARTITION BY endpoint)) AS sqrt_var
    FROM request_logs
    WHERE log_date = today()
)
WHERE abs(z_score) > 2
ORDER BY abs(z_score) DESC
LIMIT 50;
```

## Combining varSamp with Other Statistical Functions

```sql
-- Full statistical summary of CPU usage per host
SELECT
    host_name,
    count()                   AS samples,
    min(cpu_percent)          AS min_cpu,
    avg(cpu_percent)          AS mean_cpu,
    max(cpu_percent)          AS max_cpu,
    varSamp(cpu_percent)      AS variance,
    sqrt(varSamp(cpu_percent)) AS std_dev
FROM host_metrics
WHERE metric_time >= now() - INTERVAL 1 HOUR
GROUP BY host_name
ORDER BY variance DESC;
```

## Storing Variance in Materialized Views

For dashboards that need real-time variance over large windows, pre-aggregate using `AggregatingMergeTree`.

```sql
CREATE TABLE hourly_latency_stats
(
    stat_hour   DateTime,
    endpoint    String,
    lat_var     AggregateFunction(varSamp, Float64)
)
ENGINE = AggregatingMergeTree()
ORDER BY (stat_hour, endpoint);

CREATE MATERIALIZED VIEW mv_hourly_latency_stats
TO hourly_latency_stats
AS
SELECT
    toStartOfHour(timestamp)       AS stat_hour,
    endpoint,
    varSampState(response_time_ms) AS lat_var
FROM request_logs
GROUP BY stat_hour, endpoint;

-- Query merged variance
SELECT
    stat_hour,
    endpoint,
    varSampMerge(lat_var) AS latency_variance
FROM hourly_latency_stats
GROUP BY stat_hour, endpoint
ORDER BY stat_hour DESC;
```

## NULL Handling

Both `varSamp()` and `varPop()` ignore NULL values. If all values are NULL, the result is NaN.

```sql
-- varSamp returns NaN when all inputs are NULL
SELECT varSamp(toNullable(toFloat64(NULL)));

-- Safely handle potential NaN in output
SELECT
    endpoint,
    if(isNaN(varSamp(response_time_ms)), 0, varSamp(response_time_ms)) AS safe_variance
FROM request_logs
GROUP BY endpoint;
```

## Summary

`varSamp()` computes sample variance using Bessel's correction (dividing by N-1), making it appropriate for data that represents a sample from a larger population. `varPop()` divides by N and should be used only when the dataset contains the entire population. In monitoring and observability workloads, `varSamp()` is the standard choice for measuring spread in latency, error rates, and other time-series metrics. Combine it with `sqrt()` to get standard deviation, or use it in window functions for inline anomaly detection.
