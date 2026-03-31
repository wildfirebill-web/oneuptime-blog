# How to Build an APM System with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, APM, Observability, Performance, Monitoring

Description: Build an Application Performance Monitoring system with ClickHouse to store traces, metrics, and errors for real-time performance analysis.

---

Application Performance Monitoring (APM) requires storing and querying high-volume telemetry data quickly. ClickHouse's columnar compression and fast aggregations make it ideal as the backend for a custom APM platform.

## Spans Table for Distributed Tracing

```sql
CREATE TABLE apm_spans
(
    trace_id String,
    span_id String,
    parent_span_id String DEFAULT '',
    service_name LowCardinality(String),
    operation_name LowCardinality(String),
    start_time DateTime64(3),
    duration_ms Float64,
    status_code UInt8,
    error UInt8 DEFAULT 0,
    attributes Map(String, String)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(start_time)
ORDER BY (service_name, start_time, trace_id);
```

## Application Metrics Table

```sql
CREATE TABLE apm_metrics
(
    service_name LowCardinality(String),
    metric_name LowCardinality(String),
    value Float64,
    tags Map(String, String),
    recorded_at DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(recorded_at)
ORDER BY (service_name, metric_name, recorded_at);
```

## P50/P95/P99 Latency per Service

```sql
SELECT
    service_name,
    operation_name,
    count() AS request_count,
    round(quantile(0.50)(duration_ms), 2) AS p50_ms,
    round(quantile(0.95)(duration_ms), 2) AS p95_ms,
    round(quantile(0.99)(duration_ms), 2) AS p99_ms,
    round(countIf(error = 1) * 100.0 / count(), 3) AS error_rate_pct
FROM apm_spans
WHERE start_time >= now() - INTERVAL 1 HOUR
GROUP BY service_name, operation_name
ORDER BY p99_ms DESC
LIMIT 20;
```

## Error Rate Trend

```sql
SELECT
    toStartOfMinute(start_time) AS minute,
    service_name,
    count() AS total,
    countIf(error = 1) AS errors,
    round(countIf(error = 1) * 100.0 / count(), 2) AS error_pct
FROM apm_spans
WHERE start_time >= now() - INTERVAL 1 HOUR
GROUP BY minute, service_name
ORDER BY minute, service_name;
```

## Slowest Traces in the Last Hour

```sql
SELECT
    trace_id,
    service_name,
    operation_name,
    start_time,
    duration_ms
FROM apm_spans
WHERE start_time >= now() - INTERVAL 1 HOUR
  AND error = 0
ORDER BY duration_ms DESC
LIMIT 10;
```

## Throughput - Requests Per Minute

```sql
SELECT
    toStartOfMinute(start_time) AS minute,
    service_name,
    count() AS rpm
FROM apm_spans
WHERE start_time >= now() - INTERVAL 30 MINUTE
  AND parent_span_id = ''
GROUP BY minute, service_name
ORDER BY minute, service_name;
```

## Materialized View for Latency Buckets

```sql
CREATE MATERIALIZED VIEW apm_latency_summary
ENGINE = AggregatingMergeTree()
ORDER BY (service_name, operation_name, minute)
AS
SELECT
    service_name,
    operation_name,
    toStartOfMinute(start_time) AS minute,
    quantilesState(0.5, 0.95, 0.99)(duration_ms) AS latency_quantiles,
    countState() AS request_count
FROM apm_spans
GROUP BY service_name, operation_name, minute;
```

## Summary

ClickHouse provides an excellent foundation for a custom APM system. Storing spans in a MergeTree table partitioned by day enables fast latency percentile queries with `quantile`, error rate tracking with `countIf`, and throughput monitoring. Materialized views using `AggregatingMergeTree` keep dashboard queries instant regardless of ingestion volume.
