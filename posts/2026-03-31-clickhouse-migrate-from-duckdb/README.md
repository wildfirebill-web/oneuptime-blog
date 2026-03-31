# How to Migrate from DuckDB to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, DuckDB, Migration, Analytics, OLAP, Parquet

Description: Migrate from DuckDB to ClickHouse for server-side analytical workloads, covering Parquet export, schema translation, and SQL dialect differences.

---

DuckDB is ideal for local, embedded analytics - running queries directly on your laptop or inside an application. When your data grows beyond a single machine, or you need multi-user concurrent queries, ClickHouse provides horizontal scaling and a persistent server model. This guide covers moving from DuckDB to ClickHouse.

## When to Migrate

Move from DuckDB to ClickHouse when you need:
- Multi-user concurrent access
- Data larger than available RAM
- Real-time ingestion pipelines
- High-availability and replication

## Exporting Data from DuckDB

DuckDB writes Parquet natively:

```sql
COPY events TO 'events.parquet' (FORMAT 'parquet');
```

Export multiple files for large tables:

```sql
COPY (SELECT * FROM events WHERE year(created_at) = 2024)
TO 'events_2024.parquet' (FORMAT 'parquet');
```

Or export to CSV:

```sql
COPY events TO 'events.csv' (FORMAT 'csv', HEADER true);
```

## Schema Translation

DuckDB uses a PostgreSQL-like type system. Example table:

```sql
CREATE TABLE events (
    event_id   UUID DEFAULT gen_random_uuid(),
    user_id    BIGINT,
    event_type VARCHAR,
    created_at TIMESTAMPTZ,
    revenue    DECIMAL(12,4)
);
```

ClickHouse equivalent:

```sql
CREATE TABLE events (
    event_id   UUID DEFAULT generateUUIDv4(),
    user_id    UInt64,
    event_type LowCardinality(String),
    created_at DateTime64(3, 'UTC'),
    revenue    Decimal(12, 4)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id);
```

## Loading Parquet into ClickHouse

```bash
clickhouse-client \
  --query "INSERT INTO events FORMAT Parquet" \
  < events.parquet
```

Or for a directory of Parquet files, use a wildcard with the file table function:

```sql
INSERT INTO events
SELECT *
FROM file('/tmp/events_*.parquet', 'Parquet');
```

## Query Translation

DuckDB and ClickHouse are both SQL-standard databases. Most queries translate directly. Key differences:

DuckDB interval syntax:

```sql
SELECT * FROM events WHERE created_at >= now() - INTERVAL '7 days';
```

ClickHouse equivalent:

```sql
SELECT * FROM events WHERE created_at >= now() - INTERVAL 7 DAY;
```

DuckDB `date_trunc`:

```sql
SELECT date_trunc('hour', created_at) AS hour, count(*) AS cnt
FROM events GROUP BY hour ORDER BY hour;
```

ClickHouse equivalent:

```sql
SELECT toStartOfHour(created_at) AS hour, count() AS cnt
FROM events GROUP BY hour ORDER BY hour;
```

DuckDB `approx_count_distinct`:

```sql
SELECT approx_count_distinct(user_id) FROM events;
```

ClickHouse equivalent:

```sql
SELECT uniq(user_id) FROM events;
```

## Using DuckDB to Inspect ClickHouse Data

Interestingly, DuckDB can also query ClickHouse via its PostgreSQL extension, useful for validation:

```sql
-- In DuckDB
INSTALL postgres;
LOAD postgres;
ATTACH 'host=clickhouse-host port=9005 dbname=default user=default' AS ch (TYPE postgres);
SELECT count(*) FROM ch.events;
```

## Summary

Migrating from DuckDB to ClickHouse is made easy by Parquet as a shared format. Export from DuckDB, create the equivalent MergeTree table in ClickHouse, and load the Parquet files. SQL translation is minimal since both databases support modern analytical SQL.
