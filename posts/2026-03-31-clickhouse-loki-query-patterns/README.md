# How to Use ClickHouse with Loki Query Patterns

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Loki, Log Analytics, Observability, SQL

Description: Learn how to store logs in ClickHouse using Loki-inspired query patterns, enabling log line search, label filtering, and rate aggregations in SQL.

---

## ClickHouse as a Loki Alternative

Grafana Loki indexes labels and streams log lines as compressed chunks. ClickHouse stores logs as rows with full columnar indexing, making it faster for aggregations and cross-service analytics while supporting similar query patterns expressed in SQL.

## Log Table Schema

```sql
CREATE TABLE logs (
  timestamp    DateTime64(9) CODEC(Delta, ZSTD(1)),
  service      LowCardinality(String),
  level        LowCardinality(String),
  trace_id     String CODEC(ZSTD(1)),
  span_id      String CODEC(ZSTD(1)),
  message      String CODEC(ZSTD(1)),
  attributes   Map(LowCardinality(String), String) CODEC(ZSTD(1))
) ENGINE = MergeTree()
PARTITION BY toDate(timestamp)
ORDER BY (service, level, timestamp)
TTL toDate(timestamp) + INTERVAL 30 DAY
SETTINGS index_granularity = 8192
```

## Loki-Style Label Filtering (SQL Equivalent)

In Loki you write `{service="payment", level="error"}`. In ClickHouse:

```sql
SELECT timestamp, message
FROM logs
WHERE service = 'payment'
  AND level = 'error'
  AND timestamp >= now() - INTERVAL 1 HOUR
ORDER BY timestamp DESC
LIMIT 100
```

## Log Line Content Search (Equivalent to |= "pattern")

In Loki: `{service="api"} |= "timeout"`. In ClickHouse:

```sql
SELECT timestamp, service, message
FROM logs
WHERE service = 'api'
  AND timestamp >= now() - INTERVAL 30 MINUTE
  AND message ILIKE '%timeout%'
ORDER BY timestamp DESC
```

For faster substring searches, use a bloom filter index:

```sql
ALTER TABLE logs ADD INDEX idx_message message TYPE tokenbf_v1(32768, 3, 0) GRANULARITY 4
```

## Rate Aggregation (Equivalent to rate())

In Loki: `rate({service="api", level="error"}[5m])`. In ClickHouse:

```sql
SELECT
  toStartOfInterval(timestamp, INTERVAL 1 MINUTE) AS minute,
  count() AS error_count
FROM logs
WHERE service = 'api'
  AND level = 'error'
  AND timestamp >= now() - INTERVAL 30 MINUTE
GROUP BY minute
ORDER BY minute
```

## Top Error Messages

```sql
SELECT
  message,
  count() AS occurrences,
  min(timestamp) AS first_seen,
  max(timestamp) AS last_seen
FROM logs
WHERE level = 'error'
  AND timestamp >= now() - INTERVAL 24 HOUR
GROUP BY message
ORDER BY occurrences DESC
LIMIT 20
```

## Summary

ClickHouse replicates Loki's label-based stream filtering and content search patterns using SQL WHERE clauses, substring matching with ILIKE, and bloom filter indexes for fast text search. Time-bucketed aggregations with `toStartOfInterval` replace Loki's `rate()` function. The key advantage is that ClickHouse performs cross-service aggregations and joins that are difficult in Loki.
