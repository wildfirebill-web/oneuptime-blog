# How to Track API Response Times in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, API, Response Time, Latency, Monitoring

Description: Track and analyze API response times in ClickHouse with percentile queries, trend detection, and per-endpoint performance breakdowns.

---

Tracking API response times at scale requires a storage layer that can handle high ingest rates and return latency percentiles in milliseconds. ClickHouse is built for exactly this use case.

## API Request Log Table

```sql
CREATE TABLE api_requests
(
    request_id UUID DEFAULT generateUUIDv4(),
    service_name LowCardinality(String),
    endpoint LowCardinality(String),
    method LowCardinality(String),
    status_code UInt16,
    response_time_ms Float64,
    request_size_bytes UInt32 DEFAULT 0,
    response_size_bytes UInt32 DEFAULT 0,
    user_id UInt64 DEFAULT 0,
    client_ip IPv4,
    requested_at DateTime64(3)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(requested_at)
ORDER BY (service_name, endpoint, requested_at)
TTL toDate(requested_at) + INTERVAL 30 DAY;
```

## Latency Percentiles per Endpoint

```sql
SELECT
    endpoint,
    method,
    count() AS request_count,
    round(avg(response_time_ms), 2) AS avg_ms,
    round(quantile(0.50)(response_time_ms), 2) AS p50_ms,
    round(quantile(0.95)(response_time_ms), 2) AS p95_ms,
    round(quantile(0.99)(response_time_ms), 2) AS p99_ms,
    round(max(response_time_ms), 2) AS max_ms
FROM api_requests
WHERE requested_at >= now() - INTERVAL 1 HOUR
  AND status_code < 500
GROUP BY endpoint, method
ORDER BY p99_ms DESC
LIMIT 20;
```

## Error Rate by Endpoint

```sql
SELECT
    endpoint,
    method,
    count() AS total,
    countIf(status_code >= 500) AS server_errors,
    countIf(status_code BETWEEN 400 AND 499) AS client_errors,
    round(countIf(status_code >= 500) * 100.0 / count(), 3) AS server_error_pct
FROM api_requests
WHERE requested_at >= now() - INTERVAL 1 HOUR
GROUP BY endpoint, method
ORDER BY server_error_pct DESC
LIMIT 20;
```

## 5-Minute Rolling Latency Trend

```sql
SELECT
    toStartOfFiveMinutes(requested_at) AS bucket,
    endpoint,
    round(quantile(0.95)(response_time_ms), 2) AS p95_ms,
    count() AS requests
FROM api_requests
WHERE requested_at >= now() - INTERVAL 2 HOUR
  AND endpoint = '/api/v1/users'
GROUP BY bucket, endpoint
ORDER BY bucket;
```

## SLA Compliance - Requests Under 200ms

```sql
SELECT
    endpoint,
    count() AS total_requests,
    countIf(response_time_ms <= 200) AS within_sla,
    round(countIf(response_time_ms <= 200) * 100.0 / count(), 2) AS sla_compliance_pct
FROM api_requests
WHERE requested_at >= now() - INTERVAL 24 HOUR
GROUP BY endpoint
ORDER BY sla_compliance_pct ASC
LIMIT 20;
```

## Requests Per Second per Endpoint

```sql
SELECT
    toStartOfMinute(requested_at) AS minute,
    endpoint,
    round(count() / 60.0, 2) AS rps
FROM api_requests
WHERE requested_at >= now() - INTERVAL 1 HOUR
GROUP BY minute, endpoint
ORDER BY minute DESC, rps DESC
LIMIT 50;
```

## Materialized View for Real-Time Dashboard

```sql
CREATE MATERIALIZED VIEW api_latency_mv
ENGINE = AggregatingMergeTree()
ORDER BY (service_name, endpoint, method, minute)
AS
SELECT
    service_name,
    endpoint,
    method,
    toStartOfMinute(requested_at) AS minute,
    quantilesState(0.5, 0.95, 0.99)(response_time_ms) AS latency_state,
    countState() AS request_count_state,
    countIfState(status_code >= 500) AS error_count_state
FROM api_requests
GROUP BY service_name, endpoint, method, minute;
```

## Summary

ClickHouse tracks API response times efficiently with per-endpoint latency percentile queries, rolling trend windows, and SLA compliance calculations. Using `AggregatingMergeTree` materialized views and `quantilesState` keeps dashboard queries fast even with millions of requests per hour. This setup gives engineering teams real-time visibility into API performance without dedicated APM vendor costs.
