# How to Use CREATE TABLE AS SELECT in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, DDL, CTAS, CREATE TABLE

Description: Learn how to use CREATE TABLE AS SELECT (CTAS) in ClickHouse to create and populate tables from query results in one statement.

---

CREATE TABLE AS SELECT (CTAS) is a convenient DDL pattern that creates a new table and populates it with data from a SELECT query in a single statement. In ClickHouse, CTAS requires explicit engine specification because ClickHouse does not have a default storage engine fallback.

## Basic CTAS Syntax

```sql
CREATE TABLE new_table
ENGINE = MergeTree()
ORDER BY id
AS SELECT * FROM source_table;
```

The column definitions are inferred from the SELECT output, so you do not need to list columns explicitly unless you want to rename or cast them.

## Engine Clause is Required

Unlike some SQL databases, ClickHouse requires you to specify the `ENGINE` clause in CTAS. Omitting it will produce an error:

```sql
-- This will fail - no ENGINE specified
CREATE TABLE bad_example AS SELECT * FROM source_table;

-- Correct form
CREATE TABLE good_example
ENGINE = MergeTree()
ORDER BY id
AS SELECT * FROM source_table;
```

## Copying a Table Schema and Data

```sql
-- Full copy of a table
CREATE TABLE events_backup
ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id)
AS SELECT * FROM events;
```

## Copying Only Schema (No Data)

Use `WHERE 1=0` to copy schema without any rows:

```sql
CREATE TABLE events_empty
ENGINE = MergeTree()
ORDER BY (created_at, user_id)
AS SELECT * FROM events WHERE 1 = 0;
```

## Transforming Data During Creation

CTAS is useful for creating derived or aggregated tables:

```sql
-- Create a pre-aggregated summary table
CREATE TABLE daily_user_stats
ENGINE = MergeTree()
ORDER BY (day, user_id)
AS
SELECT
    toDate(created_at)    AS day,
    user_id,
    count()               AS event_count,
    countIf(type = 'purchase') AS purchase_count,
    sumIf(amount, type = 'purchase') AS total_spend
FROM events
GROUP BY day, user_id;
```

## Using IF NOT EXISTS

```sql
CREATE TABLE IF NOT EXISTS events_backup
ENGINE = MergeTree()
ORDER BY (created_at, user_id)
AS SELECT * FROM events;
```

## Selecting a Subset of Columns

You can rename columns or select a subset in the SELECT list:

```sql
CREATE TABLE events_slim
ENGINE = MergeTree()
ORDER BY event_id
AS
SELECT
    event_id,
    user_id,
    toDate(created_at) AS event_date,
    type               AS event_type
FROM events;
```

## CTAS with ReplacingMergeTree

```sql
CREATE TABLE user_latest_state
ENGINE = ReplacingMergeTree(updated_at)
ORDER BY user_id
AS
SELECT
    user_id,
    max(updated_at) AS updated_at,
    argMax(status, updated_at) AS status,
    argMax(email, updated_at)  AS email
FROM user_events
GROUP BY user_id;
```

## CTAS with AggregatingMergeTree

```sql
CREATE TABLE page_view_agg
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, page_path)
AS
SELECT
    toDate(created_at)              AS event_date,
    page_path,
    countState()                    AS views_state,
    uniqState(user_id)              AS unique_users_state
FROM page_views
GROUP BY event_date, page_path;
```

## Creating on a Remote Cluster

On distributed setups, use `ON CLUSTER`:

```sql
CREATE TABLE events_distributed ON CLUSTER my_cluster
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events_distributed', '{replica}')
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id)
AS SELECT * FROM events WHERE 1 = 0;
```

## Inserting Into an Existing Table from SELECT

When the table already exists, use `INSERT INTO ... SELECT` instead:

```sql
INSERT INTO events_backup
SELECT * FROM events
WHERE toDate(created_at) = today() - 1;
```

## Summary

CTAS in ClickHouse combines table creation and data loading in a single statement. The `ENGINE` clause is mandatory, and the column schema is inferred from the SELECT output. CTAS is well suited for creating backup copies, derived aggregations, and filtered snapshots of existing tables without writing separate CREATE TABLE and INSERT statements.
