# How to Track Service Mesh Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Service Mesh, Istio, Envoy, Traffic Analytics, Observability

Description: Learn how to store Istio/Envoy service mesh metrics in ClickHouse and query traffic rates, latency, error rates, and dependency maps.

---

Service meshes like Istio emit rich per-request telemetry - source service, destination service, response code, latency, and bytes transferred. Storing this data in ClickHouse unlocks long-term retention and flexible analysis beyond what Prometheus can provide within its default retention windows.

## Schema

```sql
CREATE TABLE mesh_requests
(
    ts               DateTime64(3),
    source_service   LowCardinality(String),
    source_namespace LowCardinality(String),
    dest_service     LowCardinality(String),
    dest_namespace   LowCardinality(String),
    method           LowCardinality(String),
    path             String,
    response_code    UInt16,
    duration_ms      UInt32,
    req_bytes        UInt32,
    resp_bytes       UInt32,
    protocol         LowCardinality(String)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (source_service, dest_service, ts);
```

## Request Rate per Service Pair

```sql
SELECT
    source_service,
    dest_service,
    count() / 60.0 AS rps
FROM mesh_requests
WHERE ts >= now() - INTERVAL 1 MINUTE
GROUP BY source_service, dest_service
ORDER BY rps DESC;
```

## Error Rate by Destination Service

```sql
SELECT
    dest_service,
    countIf(response_code >= 500) * 100.0 / count() AS error_rate_pct,
    count() AS total_requests
FROM mesh_requests
WHERE ts >= now() - INTERVAL 1 HOUR
GROUP BY dest_service
ORDER BY error_rate_pct DESC;
```

## Latency Percentiles per Route

```sql
SELECT
    dest_service,
    path,
    quantile(0.5)(duration_ms)  AS p50_ms,
    quantile(0.95)(duration_ms) AS p95_ms,
    quantile(0.99)(duration_ms) AS p99_ms
FROM mesh_requests
WHERE ts >= now() - INTERVAL 24 HOUR
  AND response_code < 500
GROUP BY dest_service, path
ORDER BY p99_ms DESC
LIMIT 20;
```

## Service Dependency Map

```sql
SELECT
    source_service,
    dest_service,
    count() AS calls,
    avg(duration_ms) AS avg_latency_ms
FROM mesh_requests
WHERE ts >= now() - INTERVAL 1 HOUR
GROUP BY source_service, dest_service
ORDER BY calls DESC;
```

## Golden Signal Dashboard Query

Combine rate, error, and duration in one query:

```sql
SELECT
    dest_service,
    count() / 3600.0                            AS rps,
    countIf(response_code >= 500) * 100.0 / count() AS error_rate_pct,
    quantile(0.99)(duration_ms)                 AS p99_ms
FROM mesh_requests
WHERE ts >= now() - INTERVAL 1 HOUR
GROUP BY dest_service
ORDER BY error_rate_pct DESC;
```

## Retry Amplification Detection

High request volumes from a single source may indicate retry storms:

```sql
SELECT
    source_service,
    dest_service,
    count() AS requests,
    countIf(response_code >= 500) AS failures,
    requests / (failures + 1) AS amplification_ratio
FROM mesh_requests
WHERE ts >= now() - INTERVAL 15 MINUTE
GROUP BY source_service, dest_service
HAVING amplification_ratio < 2 AND failures > 100
ORDER BY failures DESC;
```

## Summary

ClickHouse provides long-term storage and flexible analysis for service mesh telemetry. Compute request rates, error rates, and latency percentiles per service pair, build dependency maps, and detect retry amplification patterns. This complements Prometheus/Grafana for long-term trend analysis beyond the Prometheus retention window.
