# How to Track CDN Performance Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, CDN, Performance, Time To First Byte, Cache Hit Rate, Analytics

Description: Learn how to ingest CDN access logs into ClickHouse and query cache hit rates, TTFB, bandwidth, and edge node performance at scale.

---

CDN access logs contain rich performance data - cache status, edge location, response times, and byte counts. Centralizing these logs in ClickHouse lets you measure cache efficiency, detect edge degradation, and understand bandwidth patterns across millions of requests per minute.

## Schema for CDN Access Logs

```sql
CREATE TABLE cdn_access_logs
(
    ts             DateTime,
    edge_node      LowCardinality(String),
    cache_status   LowCardinality(String), -- 'HIT','MISS','BYPASS','EXPIRED'
    method         LowCardinality(String),
    url_path       String,
    status_code    UInt16,
    ttfb_ms        UInt32,
    bytes_sent     UInt64,
    country_code   LowCardinality(FixedString(2)),
    client_ip      IPv4
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (edge_node, ts);
```

## Cache Hit Rate by Edge Node

```sql
SELECT
    edge_node,
    countIf(cache_status = 'HIT') * 100.0 / count() AS hit_rate_pct,
    count() AS total_requests
FROM cdn_access_logs
WHERE ts >= now() - INTERVAL 1 HOUR
GROUP BY edge_node
ORDER BY hit_rate_pct ASC;
```

## TTFB Percentiles

Time-to-first-byte is a key user-experience signal:

```sql
SELECT
    edge_node,
    quantile(0.5)(ttfb_ms)  AS p50_ms,
    quantile(0.95)(ttfb_ms) AS p95_ms,
    quantile(0.99)(ttfb_ms) AS p99_ms
FROM cdn_access_logs
WHERE ts >= now() - INTERVAL 24 HOUR
  AND cache_status = 'HIT'
GROUP BY edge_node
ORDER BY p99_ms DESC;
```

## Bandwidth by Country

Identify the top bandwidth-consuming regions:

```sql
SELECT
    country_code,
    formatReadableSize(sum(bytes_sent)) AS total_bandwidth,
    sum(bytes_sent)                      AS bytes
FROM cdn_access_logs
WHERE ts >= now() - INTERVAL 7 DAY
GROUP BY country_code
ORDER BY bytes DESC
LIMIT 20;
```

## Error Rate Trend

Detect 5xx spikes across the CDN:

```sql
SELECT
    toStartOfMinute(ts) AS minute,
    countIf(status_code >= 500) * 100.0 / count() AS error_rate_pct
FROM cdn_access_logs
WHERE ts >= now() - INTERVAL 6 HOUR
GROUP BY minute
ORDER BY minute;
```

## Top Cache-Miss URLs

Find which assets are not being cached:

```sql
SELECT
    url_path,
    count() AS misses
FROM cdn_access_logs
WHERE cache_status = 'MISS'
  AND ts >= now() - INTERVAL 1 DAY
GROUP BY url_path
ORDER BY misses DESC
LIMIT 20;
```

## Materialized View for Hourly Summaries

```sql
CREATE MATERIALIZED VIEW cdn_hourly_summary_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (hour, edge_node, cache_status)
AS
SELECT
    toStartOfHour(ts)  AS hour,
    edge_node,
    cache_status,
    count()            AS requests,
    sum(bytes_sent)    AS bytes,
    sum(ttfb_ms)       AS ttfb_sum
FROM cdn_access_logs
GROUP BY hour, edge_node, cache_status;
```

## Summary

ClickHouse is a natural fit for CDN analytics. Store raw access logs in a partitioned MergeTree table, then query cache hit rates, TTFB percentiles, bandwidth distribution, and error rates using simple aggregations. Pre-aggregate hourly summaries with materialized views to power real-time dashboards without scanning billions of raw log rows.
