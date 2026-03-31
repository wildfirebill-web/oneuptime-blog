# How to Analyze Docker Registry Metrics in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Docker Registry, Metric, Container, Analytics

Description: Learn how to store and analyze Docker registry metrics in ClickHouse to monitor pull rates, push rates, image sizes, and storage trends.

---

## Why Analyze Docker Registry Metrics

Docker registries are critical infrastructure. Tracking pull rates, failed authentications, and image sizes helps you plan capacity, detect abuse, and optimize storage costs. ClickHouse handles the high event volume from busy registries with ease.

## Registry Events Table

```sql
CREATE TABLE docker_registry_events (
    event_time DateTime,
    action LowCardinality(String),  -- push, pull, delete
    repository String,
    tag String,
    image_digest String,
    image_size_bytes UInt64,
    client_ip String,
    user_agent String,
    status_code UInt16,
    duration_ms UInt32,
    bytes_transferred UInt64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (repository, action, event_time);
```

## Top Pulled Images

```sql
SELECT
    repository,
    tag,
    count() AS pull_count,
    sum(bytes_transferred) AS total_bytes
FROM docker_registry_events
WHERE action = 'pull'
  AND event_time >= now() - INTERVAL 7 DAY
GROUP BY repository, tag
ORDER BY pull_count DESC
LIMIT 20;
```

## Failed Pull Rate by Repository

```sql
SELECT
    repository,
    countIf(status_code >= 400) AS failed_pulls,
    countIf(status_code < 400) AS successful_pulls,
    countIf(status_code >= 400) * 100.0 / count() AS failure_pct
FROM docker_registry_events
WHERE action = 'pull'
  AND event_time >= now() - INTERVAL 24 HOUR
GROUP BY repository
ORDER BY failure_pct DESC;
```

## Bandwidth Usage Over Time

```sql
SELECT
    toStartOfHour(event_time) AS hour,
    action,
    sum(bytes_transferred) AS total_bytes,
    count() AS event_count
FROM docker_registry_events
WHERE event_time >= now() - INTERVAL 7 DAY
GROUP BY hour, action
ORDER BY hour DESC;
```

## Storage Growth by Repository

```sql
SELECT
    repository,
    max(image_size_bytes) AS max_image_size,
    count(DISTINCT image_digest) AS unique_images,
    sum(image_size_bytes) AS total_stored_bytes
FROM docker_registry_events
WHERE action = 'push'
GROUP BY repository
ORDER BY total_stored_bytes DESC
LIMIT 20;
```

## Detecting Unusual Activity

Find IPs with unusually high pull rates (potential abuse):

```sql
SELECT
    client_ip,
    count() AS pull_count,
    count(DISTINCT repository) AS repos_accessed
FROM docker_registry_events
WHERE action = 'pull'
  AND event_time >= now() - INTERVAL 1 HOUR
GROUP BY client_ip
HAVING pull_count > 1000
ORDER BY pull_count DESC;
```

## Hourly Materialized View

```sql
CREATE MATERIALIZED VIEW docker_registry_hourly
ENGINE = SummingMergeTree()
ORDER BY (repository, action, hour)
AS SELECT
    toStartOfHour(event_time) AS hour,
    repository,
    action,
    countState() AS event_count,
    sumState(bytes_transferred) AS bytes
FROM docker_registry_events
GROUP BY hour, repository, action;
```

## Summary

ClickHouse provides the throughput and query performance needed to analyze Docker registry metrics at scale. With a properly partitioned schema, you can monitor pull trends, detect anomalies, and plan storage capacity - turning raw registry logs into actionable infrastructure intelligence.
