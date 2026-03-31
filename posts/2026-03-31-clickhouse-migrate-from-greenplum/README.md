# How to Migrate from Greenplum to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Greenplum, Migration, Analytics, Data Warehouse, PostgreSQL

Description: Move your analytical workloads from Greenplum to ClickHouse, covering schema translation, data export via COPY, and query rewrites.

---

Greenplum is a massively parallel PostgreSQL-based data warehouse. ClickHouse offers faster aggregation queries, simpler cluster management, and lower cost for most analytics workloads. Because Greenplum uses standard PostgreSQL SQL, translation is largely mechanical.

## Key Differences

| Greenplum | ClickHouse |
|---|---|
| MPP PostgreSQL | Columnar MergeTree |
| DISTRIBUTED BY | ORDER BY (implicit sharding) |
| Append-optimized tables | MergeTree parts |
| GPFDIST / external tables | S3 / file table functions |

## Exporting Schemas from Greenplum

```bash
pg_dump -U gpadmin -s -t events mydb > events_schema.sql
```

## Translating a Greenplum Schema

Greenplum DDL:

```sql
CREATE TABLE events (
    event_id   BIGSERIAL,
    user_id    BIGINT,
    event_type VARCHAR(64),
    created_at TIMESTAMP,
    revenue    NUMERIC(12, 4)
) WITH (appendoptimized=true, orientation=column)
DISTRIBUTED BY (user_id);
```

ClickHouse equivalent:

```sql
CREATE TABLE events (
    event_id   UInt64,
    user_id    UInt64,
    event_type LowCardinality(String),
    created_at DateTime,
    revenue    Decimal(12, 4)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (user_id, created_at);
```

## Exporting Data from Greenplum

Use `COPY` to export to CSV:

```sql
COPY (SELECT * FROM events)
TO '/tmp/events.csv'
WITH (FORMAT csv, HEADER true, NULL '');
```

For large tables, use GPFDIST for parallel export:

```bash
gpfdist -d /tmp/export -p 8080 &

psql -U gpadmin -d mydb -c "
  CREATE WRITABLE EXTERNAL TABLE ext_events (LIKE events)
  LOCATION ('gpfdist://localhost:8080/events.csv')
  FORMAT 'CSV' (HEADER)
  DISTRIBUTED BY (user_id);

  INSERT INTO ext_events SELECT * FROM events;
"
```

## Loading into ClickHouse

```bash
clickhouse-client \
  --query "INSERT INTO events FORMAT CSVWithNames" \
  < /tmp/events.csv
```

Stream directly from Greenplum to ClickHouse using `psql` and `clickhouse-client`:

```bash
psql -U gpadmin -d mydb -c "COPY events TO STDOUT CSV HEADER" \
  | clickhouse-client --query "INSERT INTO events FORMAT CSVWithNames"
```

## Query Translation

Greenplum window function:

```sql
SELECT
    user_id,
    event_type,
    SUM(revenue) OVER (PARTITION BY user_id ORDER BY created_at) AS running_total
FROM events;
```

ClickHouse equivalent:

```sql
SELECT
    user_id,
    event_type,
    sum(revenue) OVER (PARTITION BY user_id ORDER BY created_at) AS running_total
FROM events;
```

Greenplum PERCENTILE_CONT:

```sql
SELECT PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY revenue) AS p95
FROM events;
```

ClickHouse equivalent:

```sql
SELECT quantile(0.95)(revenue) AS p95
FROM events;
```

## Handling SERIAL Columns

Greenplum uses `BIGSERIAL` for auto-increment. In ClickHouse, use `generateUUIDv4()` or application-generated IDs, or use `rowNumberInAllBlocks()` if order matters:

```sql
CREATE TABLE events (
    event_id   UUID DEFAULT generateUUIDv4(),
    -- ... rest of columns
) ENGINE = MergeTree() ORDER BY (created_at);
```

## Summary

Migrating from Greenplum to ClickHouse is made easier by Greenplum's PostgreSQL-compatible SQL. The `COPY` command provides a clean export path, and the streaming pipe approach eliminates intermediate files for large tables. Most analytical queries translate directly with minor function name changes.
