# How to Migrate from Apache Pinot to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Apache Pinot, Migration, Analytics, OLAP, Data Engineering

Description: Step-by-step guide to migrating your real-time analytics workloads from Apache Pinot to ClickHouse, covering schema translation and data movement.

---

Apache Pinot was designed for ultra-low-latency analytics on high-throughput event streams. ClickHouse covers the same use cases while offering a richer SQL dialect, simpler operations, and a lower resource footprint for most teams. This guide walks through the migration process.

## Comparing Key Concepts

| Apache Pinot | ClickHouse |
|---|---|
| Table (REALTIME/OFFLINE) | MergeTree table |
| Schema JSON | CREATE TABLE DDL |
| Segment | Part |
| Pinot minion | Background merges |
| Star-Tree index | Projection |

## Translating the Schema

A Pinot schema and table config:

```json
{
  "schemaName": "events",
  "dimensionFieldSpecs": [
    { "name": "user_id", "dataType": "LONG" },
    { "name": "event_type", "dataType": "STRING" }
  ],
  "metricFieldSpecs": [
    { "name": "value", "dataType": "FLOAT" }
  ],
  "dateTimeFieldSpecs": [
    { "name": "ts", "dataType": "LONG", "format": "EPOCH|MILLISECONDS|1" }
  ]
}
```

Becomes this ClickHouse DDL:

```sql
CREATE TABLE events (
    ts         DateTime64(3),
    user_id    UInt64,
    event_type LowCardinality(String),
    value      Float32
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (event_type, user_id, ts);
```

## Exporting Data from Pinot

Use the Pinot SQL query console or REST API to export segments as CSV:

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"sql": "SELECT * FROM events LIMIT 1000000"}' \
  http://pinot-broker:8099/query/sql \
  | jq -r '.resultTable.rows[] | @csv' > events.csv
```

For large tables, export in time-range batches:

```bash
START=1700000000000
END=1700086400000

curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"sql\": \"SELECT * FROM events WHERE ts BETWEEN ${START} AND ${END}\"}" \
  http://pinot-broker:8099/query/sql \
  | jq -r '.resultTable.rows[] | @csv' >> events_batch.csv
```

## Loading Data into ClickHouse

```bash
clickhouse-client \
  --query "INSERT INTO events FORMAT CSVWithNames" \
  < events.csv
```

Or use the HTTP interface for large files:

```bash
cat events.csv | curl \
  'http://localhost:8123/?query=INSERT+INTO+events+FORMAT+CSVWithNames' \
  --data-binary @-
```

## Translating Queries

Pinot query:

```sql
SELECT event_type, COUNT(*) AS cnt, SUM(value) AS total
FROM events
WHERE ts BETWEEN 1700000000000 AND 1700086400000
GROUP BY event_type
ORDER BY cnt DESC
LIMIT 10
```

ClickHouse equivalent (using `toDateTime64`):

```sql
SELECT
    event_type,
    count() AS cnt,
    sum(value) AS total
FROM events
WHERE ts BETWEEN toDateTime64(1700000000, 3) AND toDateTime64(1700086400, 3)
GROUP BY event_type
ORDER BY cnt DESC
LIMIT 10;
```

## Replacing Star-Tree Indexes with Projections

Pinot star-tree indexes pre-aggregate data for fast GROUP BY queries. ClickHouse projections serve the same purpose:

```sql
ALTER TABLE events
ADD PROJECTION agg_by_type (
    SELECT event_type, sum(value), count()
    GROUP BY event_type
);

ALTER TABLE events MATERIALIZE PROJECTION agg_by_type;
```

## Validating the Migration

```sql
-- Row counts must match
SELECT count() FROM events;

-- Spot check aggregates
SELECT event_type, sum(value) AS total
FROM events
GROUP BY event_type
ORDER BY total DESC
LIMIT 5;
```

## Summary

Migrating from Apache Pinot to ClickHouse involves translating JSON schemas to SQL DDL, exporting data as CSV in batches, and rewriting queries to use ClickHouse's richer SQL dialect. Projections can replace Star-Tree indexes for pre-aggregated workloads.
