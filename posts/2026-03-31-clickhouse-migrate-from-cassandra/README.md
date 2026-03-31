# How to Migrate from Apache Cassandra to ClickHouse for Analytics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Apache Cassandra, Migration, Analytics, NoSQL, OLAP

Description: Learn how to migrate analytical data from Apache Cassandra to ClickHouse, translating CQL schemas, exporting data, and rewriting aggregation queries.

---

Cassandra excels at high-throughput writes and key-value lookups, but analytical queries with GROUP BY and aggregations are painful. ClickHouse is purpose-built for analytics. This guide covers the migration path from Cassandra to ClickHouse for analytics use cases.

## Why Migrate for Analytics?

Cassandra struggles with analytics because:
- Full table scans require `ALLOW FILTERING`, which is slow
- No native aggregation pushdown
- Schema must be designed around query patterns, not data shape

ClickHouse resolves all of these with columnar storage and vectorized aggregation.

## Mapping Data Models

| Cassandra | ClickHouse |
|---|---|
| Keyspace | Database |
| Table | Table |
| Partition key | ORDER BY leading columns |
| Clustering key | ORDER BY trailing columns |
| TTL | `TTL` clause on MergeTree |

## Translating a Schema

Cassandra CQL:

```sql
CREATE TABLE analytics.events (
    user_id    UUID,
    event_type TEXT,
    occurred   TIMESTAMP,
    properties MAP<TEXT, TEXT>,
    PRIMARY KEY ((user_id), occurred)
) WITH CLUSTERING ORDER BY (occurred DESC)
  AND default_time_to_live = 7776000;
```

ClickHouse equivalent:

```sql
CREATE TABLE events (
    user_id    UUID,
    event_type LowCardinality(String),
    occurred   DateTime,
    properties Map(String, String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(occurred)
ORDER BY (user_id, occurred)
TTL occurred + INTERVAL 90 DAY;
```

## Exporting Data from Cassandra

Use `COPY TO` in `cqlsh`:

```bash
cqlsh -e "COPY analytics.events TO '/tmp/events.csv' WITH HEADER=TRUE AND DELIMITER=',';"
```

For large tables, export in token ranges using Spark:

```bash
spark-submit \
  --packages com.datastax.spark:spark-cassandra-connector_2.12:3.4.0 \
  export_cassandra.py
```

Sample Spark export script:

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .config("spark.cassandra.connection.host", "cassandra-host") \
    .getOrCreate()

df = spark.read \
    .format("org.apache.spark.sql.cassandra") \
    .options(table="events", keyspace="analytics") \
    .load()

df.write.csv("/tmp/events_export", header=True)
```

## Loading into ClickHouse

```bash
clickhouse-client \
  --query "INSERT INTO events FORMAT CSVWithNames" \
  < /tmp/events.csv
```

For UUID columns, make sure the export preserves the UUID string format - ClickHouse accepts standard UUID strings directly.

## Rewriting Analytical Queries

Cassandra (requires ALLOW FILTERING and is slow):

```sql
SELECT event_type, COUNT(*) FROM events
WHERE occurred > '2024-01-01'
ALLOW FILTERING;
```

ClickHouse (fast columnar scan):

```sql
SELECT event_type, count() AS total
FROM events
WHERE occurred >= '2024-01-01'
GROUP BY event_type
ORDER BY total DESC;
```

## Querying Map Columns

Cassandra map access: `properties['key']`

ClickHouse map access (same syntax):

```sql
SELECT properties['browser'] AS browser, count() AS cnt
FROM events
WHERE properties['browser'] != ''
GROUP BY browser
ORDER BY cnt DESC;
```

## Validating the Migration

```sql
-- Check row counts
SELECT count() FROM events;

-- Validate date ranges
SELECT min(occurred), max(occurred) FROM events;

-- Spot-check user data
SELECT * FROM events WHERE user_id = toUUID('some-uuid') LIMIT 10;
```

## Summary

Migrating from Cassandra to ClickHouse for analytics unlocks fast GROUP BY queries and aggregation without schema redesign workarounds. The `COPY TO` export path covers most table sizes, and Spark handles large multi-node clusters with range-based extraction.
