# How to Store Application Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Metric, Observability, Time Series, Monitoring

Description: Learn how to store and query application metrics in ClickHouse with a schema that supports counters, gauges, and histograms at scale.

---

Storing application metrics in ClickHouse gives you a cost-effective alternative to Prometheus long-term storage or commercial metric platforms. ClickHouse's time-series query patterns and compression make it well-suited for metrics at large scale.

## Metrics Table

```sql
CREATE TABLE app_metrics
(
    service_name LowCardinality(String),
    metric_name LowCardinality(String),
    metric_type LowCardinality(String),
    value Float64,
    labels Map(String, String),
    recorded_at DateTime64(3)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(recorded_at)
ORDER BY (service_name, metric_name, recorded_at)
TTL toDate(recorded_at) + INTERVAL 90 DAY;
```

## Insert Sample Metrics

```sql
INSERT INTO app_metrics VALUES
('api-service', 'http_requests_total', 'counter', 1, {'method': 'GET', 'path': '/users', 'status': '200'}, now()),
('api-service', 'http_request_duration_ms', 'gauge', 142.3, {'method': 'GET', 'path': '/users'}, now()),
('api-service', 'db_connections_active', 'gauge', 12, {}, now());
```

## Rate of Change for Counters

```sql
SELECT
    toStartOfMinute(recorded_at) AS minute,
    service_name,
    max(value) - min(value) AS delta
FROM app_metrics
WHERE metric_name = 'http_requests_total'
  AND service_name = 'api-service'
  AND recorded_at >= now() - INTERVAL 1 HOUR
GROUP BY minute, service_name
ORDER BY minute;
```

## Gauge Percentiles Over Time

```sql
SELECT
    toStartOfFiveMinutes(recorded_at) AS bucket,
    service_name,
    round(quantile(0.50)(value), 2) AS p50,
    round(quantile(0.95)(value), 2) AS p95,
    round(quantile(0.99)(value), 2) AS p99
FROM app_metrics
WHERE metric_name = 'http_request_duration_ms'
  AND recorded_at >= now() - INTERVAL 1 HOUR
GROUP BY bucket, service_name
ORDER BY bucket;
```

## Current Gauge Value per Service

```sql
SELECT
    service_name,
    metric_name,
    argMax(value, recorded_at) AS latest_value,
    max(recorded_at) AS last_updated
FROM app_metrics
WHERE metric_type = 'gauge'
  AND recorded_at >= now() - INTERVAL 5 MINUTE
GROUP BY service_name, metric_name
ORDER BY service_name, metric_name;
```

## Anomaly Detection - Values Above Threshold

```sql
WITH stats AS (
    SELECT
        service_name,
        metric_name,
        avg(value) AS mean_val,
        stddevPop(value) AS std_val
    FROM app_metrics
    WHERE recorded_at BETWEEN now() - INTERVAL 1 HOUR AND now() - INTERVAL 5 MINUTE
    GROUP BY service_name, metric_name
)
SELECT
    m.service_name,
    m.metric_name,
    m.value,
    m.recorded_at,
    s.mean_val,
    round((m.value - s.mean_val) / nullIf(s.std_val, 0), 2) AS z_score
FROM app_metrics m
JOIN stats s ON m.service_name = s.service_name AND m.metric_name = s.metric_name
WHERE m.recorded_at >= now() - INTERVAL 5 MINUTE
  AND abs((m.value - s.mean_val) / nullIf(s.std_val, 0)) > 3
ORDER BY abs(z_score) DESC;
```

## TTL and Downsampling

Keep high-resolution data for 7 days and roll up to hourly after that.

```sql
CREATE TABLE app_metrics_hourly
(
    service_name LowCardinality(String),
    metric_name LowCardinality(String),
    hour DateTime,
    avg_value Float64,
    max_value Float64,
    min_value Float64
)
ENGINE = MergeTree()
ORDER BY (service_name, metric_name, hour)
TTL hour + INTERVAL 365 DAY;

INSERT INTO app_metrics_hourly
SELECT
    service_name,
    metric_name,
    toStartOfHour(recorded_at) AS hour,
    avg(value),
    max(value),
    min(value)
FROM app_metrics
WHERE toDate(recorded_at) = yesterday()
GROUP BY service_name, metric_name, hour;
```

## Summary

ClickHouse handles application metrics storage efficiently through a flexible schema with `Map` labels, built-in TTL for automatic data expiry, and fast aggregation functions for rates, percentiles, and anomaly detection. Pairing high-resolution raw tables with downsampled hourly rollups balances storage cost against query flexibility.
