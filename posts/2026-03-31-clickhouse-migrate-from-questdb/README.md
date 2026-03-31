# How to Migrate from QuestDB to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, QuestDB, Migration, Time-Series, Analytics, OLAP

Description: Migrate time-series data from QuestDB to ClickHouse by exporting via the REST API or PostgreSQL wire protocol and translating SAMPLE BY queries.

---

QuestDB is a fast time-series database optimized for append-only ingestion. ClickHouse covers the same use cases while adding richer SQL support, better JOIN performance, and more mature clustering options. This guide walks through the migration.

## Export Options from QuestDB

QuestDB offers several export paths:

1. REST API CSV export
2. PostgreSQL wire protocol (psql-compatible)
3. ILP (InfluxDB Line Protocol) re-stream

## Option 1 - REST API Export

```bash
curl "http://questdb-host:9000/exp?query=SELECT+*+FROM+events&limit=1000000" \
  -o events.csv
```

For large tables, paginate using `LIMIT/OFFSET`:

```bash
OFFSET=0
LIMIT=100000

while true; do
  COUNT=$(curl -s "http://questdb-host:9000/exp?query=SELECT+*+FROM+events+LIMIT+${LIMIT},${OFFSET}" \
    -o "events_${OFFSET}.csv" --write-out "%{size_download}")

  [ "$COUNT" -lt 1000 ] && break
  OFFSET=$((OFFSET + LIMIT))
done
```

## Option 2 - PostgreSQL Wire Protocol

QuestDB supports the PostgreSQL wire protocol on port 8812:

```bash
psql -h questdb-host -p 8812 -U admin -d qdb \
  -c "\COPY events TO '/tmp/events.csv' CSV HEADER"
```

## Schema Translation

QuestDB schema (from `SHOW COLUMNS FROM events`):

```text
timestamp TIMESTAMP (designated)
sensor_id SYMBOL
value     DOUBLE
status    SYMBOL
```

ClickHouse equivalent:

```sql
CREATE TABLE events (
    timestamp DateTime64(6),
    sensor_id LowCardinality(String),
    value     Float64,
    status    LowCardinality(String)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (sensor_id, timestamp);
```

Note: QuestDB's `SYMBOL` type is a low-cardinality string - map it to `LowCardinality(String)` in ClickHouse.

## Loading into ClickHouse

```bash
clickhouse-client \
  --query "INSERT INTO events FORMAT CSVWithNames" \
  < events.csv
```

## Query Translation

QuestDB uses `SAMPLE BY` for time-bucket aggregation:

```sql
-- QuestDB
SELECT
    timestamp,
    sensor_id,
    avg(value) AS avg_val
FROM events
WHERE timestamp IN '2024-01-01T00:00:00Z;7d'
SAMPLE BY 1h ALIGN TO CALENDAR;
```

ClickHouse equivalent using `toStartOfHour`:

```sql
SELECT
    toStartOfHour(timestamp) AS hour,
    sensor_id,
    avg(value) AS avg_val
FROM events
WHERE timestamp >= '2024-01-01 00:00:00'
  AND timestamp  < '2024-01-08 00:00:00'
GROUP BY hour, sensor_id
ORDER BY sensor_id, hour;
```

QuestDB `LATEST ON` (last value per key):

```sql
-- QuestDB
SELECT * FROM sensors LATEST ON timestamp PARTITION BY sensor_id;
```

ClickHouse equivalent using `argMax`:

```sql
SELECT
    sensor_id,
    argMax(value, timestamp) AS latest_value,
    max(timestamp) AS latest_ts
FROM sensors
GROUP BY sensor_id;
```

## Handling Microsecond Timestamps

QuestDB uses microsecond precision. ClickHouse supports `DateTime64(6)` for the same:

```sql
CREATE TABLE events (
    timestamp DateTime64(6),
    -- ...
) ENGINE = MergeTree() ORDER BY timestamp;
```

## Summary

Migrating from QuestDB to ClickHouse is straightforward via the REST API or PostgreSQL protocol export. The main query translation challenge is replacing `SAMPLE BY` with `toStartOfHour/Minute/Day` + `GROUP BY`, and replacing `LATEST ON` with `argMax`.
