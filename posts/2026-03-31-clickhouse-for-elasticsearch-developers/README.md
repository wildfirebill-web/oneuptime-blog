# ClickHouse for Elasticsearch Developers - Key Differences

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Elasticsearch, Migration, Log Analytics, Full-Text Search

Description: A guide for Elasticsearch developers learning ClickHouse, covering key differences in indexing, query syntax, and log analytics use cases.

---

Elasticsearch is popular for log analytics and full-text search. ClickHouse can handle the same log analytics workloads at lower cost and with faster aggregations, though its full-text search capabilities differ. This guide helps Elasticsearch developers get productive in ClickHouse quickly.

## Indexing Model

Elasticsearch automatically indexes every field as an inverted index. ClickHouse uses columnar storage with an optional sparse primary index and no automatic secondary indexing. This means ClickHouse uses far less storage and memory but requires you to design your schema query-first.

```sql
-- Design ClickHouse schema for your most common query patterns
CREATE TABLE logs (
    timestamp DateTime64(3),
    level LowCardinality(String),
    service LowCardinality(String),
    message String,
    trace_id String
) ENGINE = MergeTree()
ORDER BY (service, level, timestamp)
PARTITION BY toDate(timestamp);
```

## Query Syntax Comparison

```json
// Elasticsearch: DSL with bool/filter
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "level": "error" } },
        { "range": { "timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "aggs": {
    "by_service": {
      "terms": { "field": "service" }
    }
  }
}
```

```sql
-- ClickHouse SQL equivalent
SELECT
    service,
    count() AS error_count
FROM logs
WHERE level = 'error'
  AND timestamp >= now() - INTERVAL 1 HOUR
GROUP BY service
ORDER BY error_count DESC;
```

## Full-Text Search

Elasticsearch's inverted index makes full-text search very fast. ClickHouse supports full-text search via the `LIKE` operator, `match()` with regex, and `hasToken()` with token-based indexes:

```sql
-- Basic full-text search
SELECT timestamp, message
FROM logs
WHERE message LIKE '%connection refused%'
LIMIT 100;

-- Faster with a token bloom filter index
ALTER TABLE logs
ADD INDEX idx_message_tokens message
TYPE tokenbf_v1(32768, 3, 0) GRANULARITY 4;

SELECT timestamp, message
FROM logs
WHERE hasToken(message, 'refused')
LIMIT 100;
```

## Aggregations

Elasticsearch aggregations map to SQL GROUP BY:

```sql
-- Top 10 services by error count
SELECT
    service,
    countIf(level = 'error') AS errors,
    countIf(level = 'warn') AS warnings,
    count() AS total
FROM logs
WHERE timestamp >= now() - INTERVAL 24 HOUR
GROUP BY service
ORDER BY errors DESC
LIMIT 10;
```

## Storage Cost Comparison

Elasticsearch typically stores 3-10x the raw data size across primary and replica shards plus inverted indexes. ClickHouse with ZSTD compression commonly achieves 5-10x compression of the raw data, making it 10-30x more storage-efficient for log data.

## Summary

Elasticsearch developers will find ClickHouse dramatically more efficient for aggregate queries and storage-intensive log workloads. The trade-off is that full-text search requires explicit token indexes rather than automatic inverted indexing. For structured logs where you filter on known fields, ClickHouse consistently outperforms Elasticsearch at a fraction of the cost.
