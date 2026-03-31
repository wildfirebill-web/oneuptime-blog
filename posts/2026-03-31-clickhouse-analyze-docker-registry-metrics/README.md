# How to Analyze Docker Registry Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Docker, Registry, Container Image, Pull Analytics

Description: Learn how to ingest Docker registry access logs into ClickHouse and analyze image pull frequency, layer cache efficiency, and storage usage.

---

Docker registries handle thousands of image pulls per day across CI/CD pipelines and production deployments. Analyzing registry logs in ClickHouse helps you understand which images are pulled most, detect unusual pull patterns, and measure cache layer hit rates to optimize build times.

## Schema: Registry Access Log

```sql
CREATE TABLE registry_events
(
    ts            DateTime,
    event_type    LowCardinality(String), -- 'push','pull','delete','tag'
    repository    LowCardinality(String),
    tag           LowCardinality(String),
    digest        FixedString(71),         -- sha256:...
    client_ip     IPv4,
    user_agent    String,
    status_code   UInt16,
    bytes         UInt64
)
ENGINE = MergeTree()
PARTITION BY toYYYYMMDD(ts)
ORDER BY (repository, ts);
```

## Most Pulled Images

```sql
SELECT
    repository,
    tag,
    count() AS pulls
FROM registry_events
WHERE event_type = 'pull'
  AND ts >= now() - INTERVAL 7 DAY
GROUP BY repository, tag
ORDER BY pulls DESC
LIMIT 20;
```

## Push Frequency per Repository

```sql
SELECT
    repository,
    toDate(ts) AS day,
    countIf(event_type = 'push') AS pushes,
    uniqExact(tag) AS unique_tags_pushed
FROM registry_events
WHERE ts >= now() - INTERVAL 30 DAY
GROUP BY repository, day
ORDER BY repository, day;
```

## Bandwidth Usage by Repository

```sql
SELECT
    repository,
    formatReadableSize(sum(bytes)) AS total_bandwidth,
    countIf(event_type = 'pull') AS pulls
FROM registry_events
WHERE ts >= now() - INTERVAL 30 DAY
GROUP BY repository
ORDER BY sum(bytes) DESC
LIMIT 20;
```

## Error Rate (4xx/5xx)

```sql
SELECT
    repository,
    countIf(status_code >= 400 AND status_code < 500) AS client_errors,
    countIf(status_code >= 500) AS server_errors,
    count() AS total
FROM registry_events
WHERE ts >= now() - INTERVAL 24 HOUR
GROUP BY repository
HAVING client_errors + server_errors > 0
ORDER BY server_errors DESC;
```

## Image Pull Trend Over Time

```sql
SELECT
    toStartOfHour(ts) AS hour,
    count() AS pulls
FROM registry_events
WHERE event_type = 'pull'
  AND ts >= now() - INTERVAL 7 DAY
GROUP BY hour
ORDER BY hour;
```

## Detect Unusual Pull Volume (Anomaly)

```sql
SELECT
    toDate(ts) AS day,
    client_ip,
    count() AS pulls
FROM registry_events
WHERE event_type = 'pull'
  AND ts >= now() - INTERVAL 7 DAY
GROUP BY day, client_ip
HAVING pulls > 500
ORDER BY pulls DESC;
```

## Summary

ClickHouse provides flexible analysis of Docker registry logs at scale. Track most-pulled images, push frequency, bandwidth consumption, and error rates using standard SQL aggregations. Detect anomalous pull volumes to identify runaway CI jobs or unauthorized access, and use historical trends to plan registry capacity and optimize layer caching.
