# How to Use the ELK to ClickHouse Migration Pattern

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Elasticsearch, ELK Stack, Migration, Log Analytics

Description: Migrate from ELK stack to ClickHouse for log analytics to cut storage costs and improve query performance on high-volume log data.

---

## Why Migrate from ELK to ClickHouse

The ELK stack (Elasticsearch, Logstash, Kibana) is a popular log platform but becomes expensive and operationally complex at scale. ClickHouse offers 5-10x better compression and significantly faster aggregation queries for structured logs.

## Key Differences to Understand

| Aspect | Elasticsearch | ClickHouse |
|--------|--------------|------------|
| Data model | Document (JSON) | Columnar (typed) |
| Storage | Inverted index | Compressed columns |
| Best for | Full-text search | Aggregations and filtering |
| Scaling | Horizontal sharding | Sharding + columnar compression |

## Designing the ClickHouse Log Schema

Map your Elasticsearch index mappings to a ClickHouse table. A typical log schema:

```sql
CREATE TABLE logs (
    timestamp       DateTime CODEC(Delta, ZSTD),
    level           LowCardinality(String),
    service         LowCardinality(String),
    host            LowCardinality(String),
    message         String,
    trace_id        String,
    span_id         String,
    attributes      Map(String, String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (service, level, timestamp)
TTL timestamp + INTERVAL 90 DAY DELETE;
```

## Migrating Historical Data

Export logs from Elasticsearch using `elasticdump` and load them into ClickHouse:

```bash
elasticdump \
  --input=http://es-host:9200/logs-2026.03 \
  --output=logs_2026_03.json \
  --type=data \
  --limit=10000

# Transform and load into ClickHouse
cat logs_2026_03.json | python3 es_to_ch.py | \
  clickhouse-client --query "INSERT INTO logs FORMAT JSONEachRow"
```

The transformation script maps ES `_source` fields to the ClickHouse column types.

## Updating the Log Ingestion Pipeline

Replace the Logstash output plugin with a ClickHouse HTTP insert. In Fluent Bit:

```text
[OUTPUT]
    Name  http
    Match *
    Host  ch.internal
    Port  8123
    URI   /?query=INSERT+INTO+logs+FORMAT+JSONEachRow
    Format json_lines
    Retry_Limit 5
```

## Querying Logs in ClickHouse

ClickHouse SQL replaces Kibana KQL for most log searches:

```sql
SELECT timestamp, service, level, message
FROM logs
WHERE service = 'payment-api'
  AND level = 'ERROR'
  AND timestamp >= now() - INTERVAL 1 HOUR
ORDER BY timestamp DESC
LIMIT 100;
```

Full-text search using LIKE or the `hasToken` function:

```sql
SELECT timestamp, message
FROM logs
WHERE hasToken(message, 'timeout')
  AND timestamp >= now() - INTERVAL 24 HOUR;
```

## Connecting a BI Tool

Use Grafana with the ClickHouse data source plugin to replicate Kibana dashboards. Most log aggregation panels translate directly to ClickHouse SQL.

## Summary

The ELK to ClickHouse migration pattern involves redesigning your log schema for columnar storage, exporting historical data, updating ingestion pipelines, and replacing KQL queries with SQL - yielding lower costs and faster analytics on large log volumes.
