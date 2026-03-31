# How to Migrate from Greenplum to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Greenplum, Migration, MPP, Data Warehouse, Analytics

Description: Migrate from Greenplum's MPP PostgreSQL architecture to ClickHouse for faster OLAP queries, simpler operations, and lower infrastructure cost.

---

Greenplum is a massively parallel PostgreSQL-based data warehouse. While powerful, its operational complexity and infrastructure requirements make it expensive to run. ClickHouse offers faster analytical queries with a simpler deployment model.

## Architecture Comparison

| Aspect | Greenplum | ClickHouse |
|--------|-----------|------------|
| Base | PostgreSQL (MPP) | Custom columnar engine |
| Segments | Many worker nodes | Shards + replicas |
| Query optimizer | GPORCA | Custom vectorized engine |
| Row storage | Row-oriented (with AO tables) | Columnar only |
| SQL compatibility | Full PostgreSQL | ANSI SQL + extensions |

## Step 1 - Export from Greenplum

Use the `COPY` command or `gpfdist` for parallel export:

```sql
COPY analytics.page_views
TO '/tmp/page_views.csv'
WITH (FORMAT csv, HEADER true, DELIMITER ',');
```

For parallel export with `gpfdist`:

```bash
gpfdist -d /tmp/export_dir -p 8080 &

-- In Greenplum
CREATE WRITABLE EXTERNAL TABLE ext_page_views (LIKE page_views)
LOCATION ('gpfdist://hostname:8080/page_views.csv')
FORMAT 'CSV' (HEADER);

INSERT INTO ext_page_views SELECT * FROM page_views;
```

## Step 2 - Create the ClickHouse Table

Map Greenplum's Append-Optimized columnar tables:

```sql
-- Greenplum DDL
CREATE TABLE analytics.page_views (
    event_id    UUID,
    user_id     TEXT,
    page        TEXT,
    duration    INTEGER DEFAULT 0,
    created_at  TIMESTAMP
)
WITH (appendoptimized=true, orientation=column, compresstype=zstd)
DISTRIBUTED BY (user_id)
PARTITION BY RANGE(created_at)
(PARTITION p2024 START('2024-01-01') END('2025-01-01'));
```

ClickHouse equivalent:

```sql
CREATE TABLE analytics.page_views
(
    event_id    UUID,
    user_id     String,
    page        LowCardinality(String),
    duration    UInt32 DEFAULT 0,
    created_at  DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id)
SETTINGS index_granularity = 8192;
```

## Step 3 - Load Data into ClickHouse

```bash
clickhouse-client \
  --query "INSERT INTO analytics.page_views FORMAT CSVWithNames" \
  < /tmp/page_views.csv
```

## Step 4 - Translate Greenplum SQL

```sql
-- Greenplum: interval arithmetic
SELECT created_at - INTERVAL '7 days' FROM page_views

-- ClickHouse
SELECT created_at - INTERVAL 7 DAY FROM analytics.page_views
```

```sql
-- Greenplum: generate_series
SELECT generate_series('2024-01-01'::date, '2024-01-31'::date, '1 day'::interval)

-- ClickHouse
SELECT arrayJoin(
    arrayMap(x -> toDate('2024-01-01') + x, range(31))
) AS day
```

## Step 5 - Rewrite Distribution Logic

Greenplum distributes rows across segments. In ClickHouse, you achieve distribution through sharding (for clusters) or just use a single-node deployment for most use cases:

```sql
-- For distributed tables
CREATE TABLE analytics.page_views_distributed
AS analytics.page_views
ENGINE = Distributed(my_cluster, analytics, page_views, cityHash64(user_id));
```

## Summary

Migrating from Greenplum to ClickHouse simplifies operations by removing the segment node architecture. SQL translation is mostly straightforward given Greenplum's PostgreSQL base, though ClickHouse-specific functions replace some PostgreSQL extensions. Query performance on pure analytical workloads typically improves due to ClickHouse's vectorized execution engine.
