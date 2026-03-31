# How to Track Service Mesh Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Service Mesh, Metric, Istio, Observability

Description: Learn how to store and analyze service mesh metrics from Istio or Envoy in ClickHouse for latency, error rate, and traffic analysis.

---

## Service Mesh Observability with ClickHouse

Service meshes like Istio generate rich telemetry - request counts, latencies, error rates, and more. ClickHouse is an ideal store for this data due to its columnar compression and fast aggregation.

## Mesh Metrics Table

```sql
CREATE TABLE service_mesh_metrics (
    event_time DateTime,
    source_service LowCardinality(String),
    source_namespace LowCardinality(String),
    destination_service LowCardinality(String),
    destination_namespace LowCardinality(String),
    protocol LowCardinality(String),
    response_code UInt16,
    latency_ms UInt32,
    request_bytes UInt32,
    response_bytes UInt32,
    connection_security_policy LowCardinality(String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (source_service, destination_service, event_time);
```

## Request Rate Between Services

```sql
SELECT
    source_service,
    destination_service,
    toStartOfMinute(event_time) AS minute,
    count() AS rps
FROM service_mesh_metrics
WHERE event_time >= now() - INTERVAL 1 HOUR
GROUP BY source_service, destination_service, minute
ORDER BY minute DESC, rps DESC;
```

## Error Rate by Service

```sql
SELECT
    destination_service,
    countIf(response_code >= 500) AS server_errors,
    countIf(response_code >= 400 AND response_code < 500) AS client_errors,
    count() AS total_requests,
    countIf(response_code >= 500) * 100.0 / count() AS error_rate_pct
FROM service_mesh_metrics
WHERE event_time >= now() - INTERVAL 1 HOUR
GROUP BY destination_service
ORDER BY error_rate_pct DESC;
```

## P99 Latency per Route

```sql
SELECT
    source_service,
    destination_service,
    quantile(0.50)(latency_ms) AS p50_ms,
    quantile(0.95)(latency_ms) AS p95_ms,
    quantile(0.99)(latency_ms) AS p99_ms
FROM service_mesh_metrics
WHERE event_time >= now() - INTERVAL 1 HOUR
  AND response_code < 500
GROUP BY source_service, destination_service
ORDER BY p99_ms DESC
LIMIT 20;
```

## mTLS Coverage

```sql
SELECT
    connection_security_policy,
    count() AS request_count,
    count() * 100.0 / sum(count()) OVER () AS pct
FROM service_mesh_metrics
WHERE event_time >= now() - INTERVAL 1 HOUR
GROUP BY connection_security_policy;
```

## Traffic Volume Heatmap

```sql
SELECT
    source_service,
    destination_service,
    sum(request_bytes + response_bytes) AS total_bytes
FROM service_mesh_metrics
WHERE event_time >= now() - INTERVAL 24 HOUR
GROUP BY source_service, destination_service
ORDER BY total_bytes DESC
LIMIT 30;
```

## Summary

ClickHouse is a powerful backend for service mesh observability. Its columnar storage and fast aggregations let you query millions of mesh metrics in milliseconds, giving platform teams real-time visibility into service dependencies, latency profiles, and error rates across the entire mesh.
