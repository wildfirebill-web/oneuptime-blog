# How to Migrate from CrateDB to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, CrateDB, Migration, Analytics, OLAP, IoT

Description: Migrate your analytical workloads from CrateDB to ClickHouse, covering schema translation, COPY TO export, and CrateDB SQL to ClickHouse SQL mapping.

---

CrateDB is a distributed SQL database built for IoT and log analytics. ClickHouse covers the same use cases with better compression, faster aggregations, and more mature SQL analytics features. Both support standard SQL, making the migration accessible.

## Architecture Comparison

| CrateDB | ClickHouse |
|---|---|
| Distributed Lucene shards | MergeTree parts |
| `COPY TO` for export | N/A (is the destination) |
| `OBJECT` type (JSON) | `Map` or `JSON` type |
| `TIMESTAMP WITH TIME ZONE` | `DateTime64` |

## Exporting Data from CrateDB

Use `COPY TO` to export to CSV or JSON:

```sql
COPY events TO '/tmp/events.csv'
WITH (format='csv', compression='none');
```

Or to S3-compatible storage:

```sql
COPY events TO 's3://my-bucket/cratedb-export/events'
WITH (format='json', compression='gzip');
```

## Schema Translation

CrateDB DDL:

```sql
CREATE TABLE events (
    event_id   TEXT PRIMARY KEY,
    user_id    BIGINT,
    event_type TEXT,
    occurred   TIMESTAMP WITH TIME ZONE,
    properties OBJECT(DYNAMIC),
    value      DOUBLE PRECISION
) CLUSTERED INTO 4 SHARDS;
```

ClickHouse equivalent:

```sql
CREATE TABLE events (
    event_id   String,
    user_id    UInt64,
    event_type LowCardinality(String),
    occurred   DateTime64(3, 'UTC'),
    properties Map(String, String),
    value      Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(occurred)
ORDER BY (occurred, user_id);
```

## Loading Data into ClickHouse

From a CSV export:

```bash
clickhouse-client \
  --query "INSERT INTO events FORMAT CSVWithNames" \
  < /tmp/events.csv
```

From JSON Lines:

```bash
clickhouse-client \
  --query "INSERT INTO events FORMAT JSONEachRow" \
  < /tmp/events.jsonl
```

## Query Translation

CrateDB query:

```sql
SELECT
    date_trunc('hour', occurred) AS hour,
    event_type,
    count(*) AS cnt,
    avg(value) AS avg_val
FROM events
WHERE occurred > NOW() - '7 days'::INTERVAL
GROUP BY 1, 2
ORDER BY cnt DESC
LIMIT 20;
```

ClickHouse equivalent:

```sql
SELECT
    toStartOfHour(occurred) AS hour,
    event_type,
    count() AS cnt,
    avg(value) AS avg_val
FROM events
WHERE occurred > now() - INTERVAL 7 DAY
GROUP BY hour, event_type
ORDER BY cnt DESC
LIMIT 20;
```

## Handling OBJECT Columns

CrateDB's `OBJECT` type stores dynamic JSON. Query nested fields with bracket notation:

```sql
-- CrateDB
SELECT properties['browser'] AS browser FROM events;
```

ClickHouse `Map` type uses the same syntax:

```sql
SELECT properties['browser'] AS browser FROM events;
```

For deeply nested objects, consider flattening to individual columns during migration for better query performance.

## Handling Arrays

CrateDB arrays map to ClickHouse `Array`:

```sql
-- CrateDB
CREATE TABLE page_views (tags ARRAY(TEXT));

-- ClickHouse
CREATE TABLE page_views (tags Array(String)) ENGINE = MergeTree() ORDER BY tuple();
```

## Summary

Migrating from CrateDB to ClickHouse is relatively smooth due to their shared SQL foundations. CrateDB's `COPY TO` provides a clean export path, and the main translation challenges are date functions and object/JSON column handling.
