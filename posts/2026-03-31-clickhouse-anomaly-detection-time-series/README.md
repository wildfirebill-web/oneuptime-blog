# How to Detect Anomalies in Time-Series Data with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Anomaly Detection, Time-Series, Observability, Analytics

Description: Use ClickHouse window functions, z-scores, and moving averages to detect statistical anomalies in time-series data like metrics, logs, and sensor readings.

## Introduction

Anomaly detection in time-series data is one of the most common and important tasks in operations, monitoring, and IoT. An anomaly might be a spike in API error rates, an unusual drop in revenue per minute, or a temperature reading that falls outside the expected range for that time of day.

ClickHouse does not ship a built-in anomaly detection algorithm, but its window functions, quantile functions, and array processing capabilities are powerful enough to implement the most effective statistical methods: z-scores, moving averages with standard deviation bands, IQR-based outlier detection, and seasonal decomposition.

## Sample Data: Server Metrics

```sql
CREATE TABLE server_metrics
(
    server_id   LowCardinality(String),
    recorded_at DateTime,
    cpu_pct     Float32,
    memory_pct  Float32,
    latency_ms  Float32,
    error_rate  Float32
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(recorded_at)
ORDER BY (server_id, recorded_at);
```

Insert some sample data with injected anomalies:

```sql
INSERT INTO server_metrics SELECT
    'web-01',
    now() - (number * 60),
    20 + rand() % 30 + if(number IN (50, 100, 150), 60, 0),
    50 + rand() % 20,
    100 + rand() % 50 + if(number IN (100), 500, 0),
    0.01 + (rand() % 5) / 1000.0
FROM numbers(500);
```

## Method 1: Z-Score Based Anomaly Detection

The z-score measures how many standard deviations a point is from the mean. A z-score above 3 (or below -3) typically signals an anomaly.

```sql
WITH stats AS (
    SELECT
        server_id,
        avg(cpu_pct)        AS mean_cpu,
        stddevPop(cpu_pct)  AS std_cpu
    FROM server_metrics
    WHERE recorded_at >= now() - INTERVAL 24 HOUR
    GROUP BY server_id
)
SELECT
    m.server_id,
    m.recorded_at,
    m.cpu_pct,
    s.mean_cpu,
    s.std_cpu,
    abs(m.cpu_pct - s.mean_cpu) / nullIf(s.std_cpu, 0) AS z_score,
    z_score > 3 AS is_anomaly
FROM server_metrics AS m
JOIN stats AS s ON m.server_id = s.server_id
WHERE m.recorded_at >= now() - INTERVAL 24 HOUR
  AND z_score > 3
ORDER BY z_score DESC;
```

## Method 2: Moving Average with Standard Deviation Bands

A rolling window approach catches local anomalies even when the global mean is not representative (e.g., metrics that trend upward):

```sql
SELECT
    server_id,
    recorded_at,
    cpu_pct,
    avg(cpu_pct) OVER w              AS rolling_mean,
    stddevPop(cpu_pct) OVER w        AS rolling_std,
    avg(cpu_pct) OVER w + 3 * stddevPop(cpu_pct) OVER w AS upper_band,
    avg(cpu_pct) OVER w - 3 * stddevPop(cpu_pct) OVER w AS lower_band,
    cpu_pct > avg(cpu_pct) OVER w + 3 * stddevPop(cpu_pct) OVER w OR
    cpu_pct < avg(cpu_pct) OVER w - 3 * stddevPop(cpu_pct) OVER w AS is_anomaly
FROM server_metrics
WHERE server_id = 'web-01'
  AND recorded_at >= now() - INTERVAL 6 HOUR
WINDOW w AS (
    PARTITION BY server_id
    ORDER BY recorded_at
    ROWS BETWEEN 30 PRECEDING AND CURRENT ROW
)
ORDER BY recorded_at;
```

## Method 3: IQR-Based Outlier Detection

The interquartile range (IQR) method is more robust than z-scores when data contains extreme outliers:

```sql
WITH iqr AS (
    SELECT
        server_id,
        quantile(0.25)(latency_ms) AS q1,
        quantile(0.75)(latency_ms) AS q3,
        q3 - q1                    AS iqr
    FROM server_metrics
    WHERE recorded_at >= now() - INTERVAL 1 HOUR
    GROUP BY server_id
)
SELECT
    m.server_id,
    m.recorded_at,
    m.latency_ms,
    i.q1,
    i.q3,
    i.iqr,
    i.q1 - 1.5 * i.iqr AS lower_fence,
    i.q3 + 1.5 * i.iqr AS upper_fence,
    m.latency_ms > i.q3 + 1.5 * i.iqr OR
    m.latency_ms < i.q1 - 1.5 * i.iqr AS is_outlier
FROM server_metrics AS m
JOIN iqr AS i ON m.server_id = i.server_id
WHERE m.recorded_at >= now() - INTERVAL 1 HOUR
  AND is_outlier = 1
ORDER BY m.latency_ms DESC;
```

## Method 4: Seasonal Decomposition - Hour-of-Day Baseline

CPU and latency patterns often have strong diurnal patterns. Comparing each reading against the historical mean for that hour-of-day removes seasonal effects:

```sql
WITH hourly_baseline AS (
    SELECT
        server_id,
        toHour(recorded_at)     AS hour_of_day,
        avg(cpu_pct)            AS expected_cpu,
        stddevPop(cpu_pct)      AS std_cpu
    FROM server_metrics
    WHERE recorded_at >= now() - INTERVAL 14 DAY
      AND recorded_at < now() - INTERVAL 1 DAY
    GROUP BY server_id, hour_of_day
)
SELECT
    m.server_id,
    m.recorded_at,
    m.cpu_pct,
    b.expected_cpu,
    b.std_cpu,
    abs(m.cpu_pct - b.expected_cpu) / nullIf(b.std_cpu, 0) AS seasonal_z_score,
    seasonal_z_score > 3 AS is_anomaly
FROM server_metrics AS m
JOIN hourly_baseline AS b
    ON m.server_id = b.server_id
    AND toHour(m.recorded_at) = b.hour_of_day
WHERE m.recorded_at >= now() - INTERVAL 3 HOUR
  AND seasonal_z_score > 3
ORDER BY seasonal_z_score DESC;
```

## Method 5: Rate-of-Change Anomaly Detection

Sometimes the absolute value is normal but the speed of change is not. A sudden jump in error rate, even if still below threshold, can predict an incident:

```sql
SELECT
    server_id,
    recorded_at,
    error_rate,
    lagInFrame(error_rate) OVER w  AS prev_error_rate,
    error_rate - lagInFrame(error_rate) OVER w AS delta,
    abs(error_rate - lagInFrame(error_rate) OVER w) > 0.05 AS spike_detected
FROM server_metrics
WHERE server_id = 'web-01'
  AND recorded_at >= now() - INTERVAL 1 HOUR
WINDOW w AS (
    PARTITION BY server_id
    ORDER BY recorded_at
    ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
)
ORDER BY recorded_at;
```

## Alerting with Materialized Views

Create a materialized view that continuously writes anomalies to an alert table as new data arrives:

```sql
CREATE TABLE metric_anomalies
(
    server_id    LowCardinality(String),
    recorded_at  DateTime,
    metric_name  LowCardinality(String),
    metric_value Float32,
    z_score      Float32,
    detected_at  DateTime DEFAULT now()
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(recorded_at)
ORDER BY (server_id, metric_name, recorded_at);
```

```sql
CREATE MATERIALIZED VIEW metric_anomalies_mv TO metric_anomalies
AS
WITH stats AS (
    SELECT
        server_id,
        avg(cpu_pct)        AS mean_cpu,
        stddevPop(cpu_pct)  AS std_cpu
    FROM server_metrics
    WHERE recorded_at >= now() - INTERVAL 1 HOUR
    GROUP BY server_id
)
SELECT
    m.server_id,
    m.recorded_at,
    'cpu_pct'                                                    AS metric_name,
    m.cpu_pct                                                    AS metric_value,
    abs(m.cpu_pct - s.mean_cpu) / nullIf(s.std_cpu, 0)          AS z_score
FROM server_metrics AS m
JOIN stats AS s ON m.server_id = s.server_id
WHERE z_score > 3;
```

## Summarizing Anomalies Over Time

```sql
SELECT
    server_id,
    metric_name,
    toDate(recorded_at)   AS day,
    count()               AS anomaly_count,
    avg(z_score)          AS avg_z_score,
    max(z_score)          AS max_z_score
FROM metric_anomalies
WHERE recorded_at >= now() - INTERVAL 30 DAY
GROUP BY server_id, metric_name, day
ORDER BY day DESC, anomaly_count DESC;
```

## Performance Tips

For large time-series tables, pre-aggregate statistics into a separate table that gets rebuilt nightly. This avoids scanning weeks of raw data every time you run an anomaly detection query:

```sql
CREATE TABLE metric_baselines
(
    server_id    LowCardinality(String),
    metric_name  LowCardinality(String),
    hour_of_day  UInt8,
    day_of_week  UInt8,
    mean         Float64,
    std_dev      Float64,
    p05          Float64,
    p95          Float64,
    computed_at  DateTime DEFAULT now()
)
ENGINE = ReplacingMergeTree(computed_at)
ORDER BY (server_id, metric_name, hour_of_day, day_of_week);
```

## Conclusion

ClickHouse provides the building blocks for effective time-series anomaly detection without requiring a dedicated ML platform. Z-scores, rolling statistics, IQR fences, and seasonal baselines cover the vast majority of production alerting use cases. Materialized views make it possible to detect anomalies in near real time as data arrives, feeding alert pipelines or downstream dashboards automatically.

**Related Reading:**

- [How to Build a Real-Time Fraud Detection System with ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-real-time-fraud-detection/view)
- [What Is Granule in ClickHouse and Why It Matters for Performance](https://oneuptime.com/blog/post/2026-03-31-clickhouse-granule-performance/view)
- [What Is ClickHouse MergeTree and How It Works](https://oneuptime.com/blog/post/2026-03-31-clickhouse-mergetree-explained/view)
