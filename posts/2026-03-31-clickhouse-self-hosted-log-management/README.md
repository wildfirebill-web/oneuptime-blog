# How to Build a Self-Hosted Log Management Platform with ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Log Management, Self-Hosted, Observability, Grafana

Description: Build a self-hosted log management platform with ClickHouse and Grafana to replace Datadog Logs or Splunk at a fraction of the cost.

---

## Why Self-Host Log Management

Log management SaaS tools can be the single largest observability cost. Datadog and Splunk charge by ingestion volume - at 10 TB/day, monthly costs can exceed $100,000. Self-hosting with ClickHouse reduces storage costs by 10-20x while giving you full query flexibility.

## Architecture

```text
Applications --> Fluent Bit / Vector --> ClickHouse --> Grafana
                                              |
                                         OneUptime (alerting)
```

## Log Storage Schema

```sql
CREATE TABLE logs.raw (
    timestamp    DateTime64(9) CODEC(Delta, ZSTD),
    level        LowCardinality(String),
    service      LowCardinality(String),
    namespace    LowCardinality(String),
    pod          String,
    trace_id     String CODEC(ZSTD),
    span_id      String CODEC(ZSTD),
    message      String CODEC(ZSTD(6)),
    labels       Map(String, String) CODEC(ZSTD)
) ENGINE = MergeTree()
PARTITION BY toDate(timestamp)
ORDER BY (service, level, timestamp)
TTL toDate(timestamp) + INTERVAL 30 DAY DELETE
SETTINGS index_granularity = 8192;
```

Add a token bloom filter for fast full-text search:

```sql
ALTER TABLE logs.raw
    ADD INDEX idx_message_tokens message
    TYPE tokenbf_v1(32768, 3, 0) GRANULARITY 4;
```

## Fluent Bit Configuration

Ship Kubernetes pod logs directly to ClickHouse:

```text
[SERVICE]
    Flush        5
    Log_Level    info

[INPUT]
    Name         tail
    Path         /var/log/containers/*.log
    Parser       docker
    Tag          kube.*

[FILTER]
    Name         kubernetes
    Match        kube.*
    Merge_Log    On

[OUTPUT]
    Name         http
    Match        *
    Host         ch.internal
    Port         8123
    URI          /?query=INSERT%20INTO%20logs.raw%20FORMAT%20JSONEachRow
    Format       json_lines
    Retry_Limit  3
```

## Log Search Queries

Search by keyword using token bloom filter:

```sql
SELECT timestamp, service, level, message
FROM logs.raw
WHERE hasToken(message, 'OutOfMemoryError')
  AND timestamp >= now() - INTERVAL 24 HOUR
ORDER BY timestamp DESC
LIMIT 100;
```

Error summary by service:

```sql
SELECT
    service,
    count() AS error_count,
    min(timestamp) AS first_seen,
    max(timestamp) AS last_seen
FROM logs.raw
WHERE level = 'ERROR'
  AND timestamp >= now() - INTERVAL 1 HOUR
GROUP BY service
ORDER BY error_count DESC;
```

## Grafana Log Panel

Configure Grafana with the ClickHouse data source and use the Logs panel type. Your query:

```sql
SELECT timestamp AS time, level, service, message
FROM logs.raw
WHERE timestamp >= $__fromTime
  AND timestamp <= $__toTime
  AND service = '$service'
ORDER BY timestamp DESC
LIMIT 1000
```

## Cost Comparison

```text
Daily ingestion: 5 TB/day
Datadog Logs: ~$30,000/month
Self-hosted ClickHouse (3 nodes, 12 TB SSD): ~$1,500/month
Savings: ~$28,500/month
```

## Summary

A self-hosted log management platform built on ClickHouse and Grafana delivers full-text log search, aggregation analytics, and long-term retention at 10-20x lower cost than commercial SaaS log management solutions.
