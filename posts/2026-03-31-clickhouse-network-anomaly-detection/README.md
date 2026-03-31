# How to Build Real-Time Network Anomaly Detection with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Anomaly Detection, Network Monitoring, Telecom, Real-Time Analytics

Description: Build real-time network anomaly detection with ClickHouse by comparing live metrics against rolling baselines to surface outliers instantly.

---

Network anomalies - traffic spikes, unusual call failure rates, sudden latency increases - need to be detected within minutes to minimize customer impact. ClickHouse enables real-time anomaly detection by comparing live metrics against rolling statistical baselines.

## Network Metrics Table

```sql
CREATE TABLE network_metrics (
    timestamp       DateTime,
    node_id         String,
    region          LowCardinality(String),
    metric_name     LowCardinality(String),
    metric_value    Float64,
    date            Date DEFAULT toDate(timestamp)
) ENGINE = MergeTree()
PARTITION BY date
ORDER BY (node_id, metric_name, timestamp);
```

## Rolling Baseline Calculation

Compute the mean and standard deviation over the previous 7 days for the same hour:

```sql
SELECT
    node_id,
    metric_name,
    avg(metric_value) AS baseline_mean,
    stddevPop(metric_value) AS baseline_stddev
FROM network_metrics
WHERE date BETWEEN today() - 8 AND today() - 1
  AND toHour(timestamp) = toHour(now())
GROUP BY node_id, metric_name;
```

## Anomaly Scoring

Flag metrics that are more than 3 standard deviations from the baseline:

```sql
WITH baseline AS (
    SELECT
        node_id,
        metric_name,
        avg(metric_value) AS mu,
        stddevPop(metric_value) AS sigma
    FROM network_metrics
    WHERE date BETWEEN today() - 8 AND today() - 1
      AND toHour(timestamp) = toHour(now())
    GROUP BY node_id, metric_name
),
current AS (
    SELECT
        node_id,
        metric_name,
        avg(metric_value) AS current_value
    FROM network_metrics
    WHERE timestamp >= now() - INTERVAL 5 MINUTE
    GROUP BY node_id, metric_name
)
SELECT
    c.node_id,
    c.metric_name,
    c.current_value,
    b.mu AS expected,
    round((c.current_value - b.mu) / b.sigma, 2) AS z_score
FROM current AS c
JOIN baseline AS b ON c.node_id = b.node_id AND c.metric_name = b.metric_name
WHERE abs((c.current_value - b.mu) / b.sigma) > 3
ORDER BY abs((c.current_value - b.mu) / b.sigma) DESC;
```

## Traffic Volume Spike Detection

Detect nodes with abnormally high call volume:

```sql
SELECT
    node_id,
    count() AS current_calls,
    round(avg(count()) OVER (ORDER BY toStartOfHour(timestamp) ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING), 0) AS baseline_calls
FROM network_metrics
WHERE metric_name = 'call_attempts'
  AND date >= today() - 7
GROUP BY node_id, toStartOfHour(timestamp)
HAVING current_calls > baseline_calls * 1.5
ORDER BY current_calls DESC;
```

## Sustained Degradation Alert

Identify nodes where a metric has been elevated for multiple consecutive 5-minute windows:

```sql
SELECT
    node_id,
    metric_name,
    count() AS consecutive_high_windows
FROM (
    SELECT
        node_id,
        metric_name,
        toStartOfFiveMinute(timestamp) AS window,
        avg(metric_value) AS window_avg
    FROM network_metrics
    WHERE timestamp >= now() - INTERVAL 1 HOUR
    GROUP BY node_id, metric_name, window
)
WHERE window_avg > 0.1
GROUP BY node_id, metric_name
HAVING consecutive_high_windows >= 6
ORDER BY consecutive_high_windows DESC;
```

## Summary

ClickHouse enables real-time anomaly detection by joining live metric windows with historical baselines computed in SQL. Z-score thresholding and consecutive-window checks provide both instant and sustained anomaly detection without the need for dedicated machine learning infrastructure.
