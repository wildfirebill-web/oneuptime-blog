# ClickHouse for Cassandra Developers - Key Differences

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cassandra, Migration, Analytics, Wide Column

Description: A guide for Cassandra developers learning ClickHouse, comparing data modeling, query patterns, and when to choose each database.

---

Cassandra and ClickHouse both handle large-scale data, but their strengths are different. Cassandra excels at high-throughput OLTP with predictable single-partition reads and writes. ClickHouse excels at analytical queries that aggregate across billions of rows. Many teams use both: Cassandra for operational data and ClickHouse for analytics.

## Data Modeling Philosophy

Cassandra data modeling is query-driven: you design tables around your specific access patterns because full table scans are expensive. ClickHouse is also query-driven in terms of sorting key design, but it supports full table scans efficiently thanks to columnar storage.

```sql
-- Cassandra: partition key must match query pattern exactly
CREATE TABLE events_by_user (
    user_id UUID,
    event_time TIMESTAMP,
    event_type TEXT,
    PRIMARY KEY (user_id, event_time)
) WITH CLUSTERING ORDER BY (event_time DESC);

-- ClickHouse: sorting key enables range scans
CREATE TABLE events (
    user_id UInt32,
    event_time DateTime,
    event_type LowCardinality(String)
) ENGINE = MergeTree()
ORDER BY (user_id, event_time);
```

## Write Path

Cassandra writes go to a commit log and memtable, then flush to SSTables. ClickHouse writes go to in-memory buffers and flush to data parts that are later merged. Both are append-optimized.

```sql
-- ClickHouse: efficient bulk insert
INSERT INTO events
SELECT * FROM generateRandom(
    'user_id UInt32, event_time DateTime, event_type String',
    42, 1000, 1
) LIMIT 1000000;
```

## Aggregation Queries

Cassandra is not designed for aggregation queries. Every aggregation requires scanning potentially all partitions. ClickHouse is purpose-built for aggregations:

```sql
-- This query is trivial in ClickHouse, expensive in Cassandra
SELECT
    event_type,
    toDate(event_time) AS day,
    count() AS occurrences,
    uniq(user_id) AS unique_users
FROM events
WHERE event_time >= now() - INTERVAL 30 DAY
GROUP BY event_type, day
ORDER BY day, occurrences DESC;
```

## Consistency Model

Cassandra uses tunable consistency (ONE, QUORUM, ALL) with eventual consistency as the default. ClickHouse replication is asynchronous by default, meaning replicas can lag. For strong consistency, use synchronous replication settings.

## Secondary Indexes

Cassandra secondary indexes are often discouraged for high-cardinality fields. ClickHouse skip indexes speed up queries without full secondary indexing:

```sql
ALTER TABLE events
ADD INDEX idx_event_type event_type
TYPE bloom_filter GRANULARITY 4;

OPTIMIZE TABLE events FINAL;
```

## TTL for Data Expiry

Both databases support TTL. In ClickHouse:

```sql
CREATE TABLE events (
    event_time DateTime,
    user_id UInt32,
    event_type String
) ENGINE = MergeTree()
ORDER BY (user_id, event_time)
TTL event_time + INTERVAL 90 DAY DELETE;
```

## Summary

Cassandra developers transitioning to ClickHouse will find the biggest difference in aggregation capabilities: ClickHouse can answer analytical queries across billions of rows in seconds without special data modeling. For write-heavy OLTP workloads with predictable single-partition access patterns, Cassandra remains the better choice.
