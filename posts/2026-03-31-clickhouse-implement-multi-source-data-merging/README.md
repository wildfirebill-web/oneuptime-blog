# How to Implement Multi-Source Data Merging in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Data Merging, ETL, Multi-Source, Ingestion, CollapsingMergeTree

Description: Learn how to merge data from multiple sources into ClickHouse using staging tables, materialized views, and merge engine patterns.

---

Production data pipelines often pull from multiple sources - different databases, Kafka topics, S3 buckets, or APIs. ClickHouse provides several patterns to merge these streams into a unified analytical dataset.

## Common Multi-Source Scenarios

```text
1. Combining Kafka streams from different services
2. Merging CDC events from multiple databases
3. Joining S3 batch loads with streaming data
4. Aggregating from multiple tenants or regions
```

## Pattern 1: Staging Tables per Source

Each source writes to its own staging table. A materialized view merges them into a target:

```sql
CREATE TABLE events_source_a (
    event_id String,
    source String DEFAULT 'source_a',
    timestamp DateTime,
    payload String
) ENGINE = MergeTree()
ORDER BY (timestamp, event_id);

CREATE TABLE events_source_b (
    event_id String,
    source String DEFAULT 'source_b',
    timestamp DateTime,
    payload String
) ENGINE = MergeTree()
ORDER BY (timestamp, event_id);

-- Unified target
CREATE TABLE events_unified (
    event_id String,
    source String,
    timestamp DateTime,
    payload String
) ENGINE = MergeTree()
ORDER BY (source, timestamp, event_id);
```

Insert into the unified table from both sources:

```sql
INSERT INTO events_unified SELECT * FROM events_source_a;
INSERT INTO events_unified SELECT * FROM events_source_b;
```

## Pattern 2: Kafka Multi-Topic Ingestion

Use separate Kafka engine tables per topic, then merge with a materialized view:

```sql
CREATE TABLE kafka_topic_a (event_id String, value Float64)
ENGINE = Kafka SETTINGS kafka_topic_list = 'topic-a', kafka_format = 'JSONEachRow',
kafka_broker_list = 'broker:9092', kafka_group_name = 'ch-group-a';

CREATE TABLE kafka_topic_b (event_id String, value Float64)
ENGINE = Kafka SETTINGS kafka_topic_list = 'topic-b', kafka_format = 'JSONEachRow',
kafka_broker_list = 'broker:9092', kafka_group_name = 'ch-group-b';

CREATE MATERIALIZED VIEW mv_topic_a TO events_unified AS
SELECT event_id, 'topic-a' AS source, value FROM kafka_topic_a;

CREATE MATERIALIZED VIEW mv_topic_b TO events_unified AS
SELECT event_id, 'topic-b' AS source, value FROM kafka_topic_b;
```

## Pattern 3: Merge Engine

The Merge engine provides a virtual table that queries multiple tables simultaneously:

```sql
CREATE TABLE events_all AS events_source_a
ENGINE = Merge(currentDatabase(), '^events_source_');
```

Query `events_all` to transparently read from all matching tables:

```sql
SELECT source, count() FROM events_all GROUP BY source;
```

## Pattern 4: S3 + Streaming Merge

Combine historical S3 data with live Kafka data:

```sql
SELECT event_id, payload FROM s3('s3://my-bucket/events/*.parquet', 'Parquet')
UNION ALL
SELECT event_id, payload FROM events_live
WHERE timestamp >= today();
```

## Handling Conflicts

When the same event_id appears in multiple sources, use `ReplacingMergeTree` to keep the latest version:

```sql
CREATE TABLE events_merged (
    event_id String,
    source String,
    timestamp DateTime,
    payload String,
    version UInt64
) ENGINE = ReplacingMergeTree(version)
ORDER BY event_id;
```

## Summary

ClickHouse supports multi-source data merging through staging tables, the Merge engine, materialized views, and Kafka multi-topic ingestion. The right pattern depends on whether your sources are batch or streaming, and whether you need conflict resolution between sources.
