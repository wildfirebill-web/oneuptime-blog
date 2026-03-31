# How to Use ClickHouse as a PostgreSQL Foreign Data Wrapper

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, PostgreSQL, Foreign Data Wrapper, Data Engineering, Analytics

Description: Learn how to expose ClickHouse tables to PostgreSQL as foreign tables using the clickhouse_fdw extension, enabling SQL joins across both databases.

---

> The clickhouse_fdw extension lets PostgreSQL query ClickHouse tables as foreign tables, enabling cross-database joins without moving data.

A PostgreSQL Foreign Data Wrapper (FDW) lets you query external data sources using standard SQL from PostgreSQL. The `clickhouse_fdw` extension exposes ClickHouse tables as foreign tables inside PostgreSQL, so your application can run queries that span both databases. This is especially useful for enriching transactional data with pre-aggregated ClickHouse analytics.

---

## Installing clickhouse_fdw

Install the extension on your PostgreSQL server.

```bash
# Prerequisites: PostgreSQL development headers
sudo apt-get install -y postgresql-server-dev-16 libcurl4-openssl-dev

# Clone and build clickhouse_fdw
git clone https://github.com/adjust/clickhouse_fdw.git
cd clickhouse_fdw
make
sudo make install

# Alternatively, use the prebuilt package for Ubuntu/Debian
curl -fsSL https://packages.clickhouse.com/deb/clickhouse.gpg \
  | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/clickhouse.gpg

sudo apt-get install -y clickhouse-fdw
```

## Loading the Extension in PostgreSQL

Enable the FDW and create the server connection.

```sql
-- Load the extension
CREATE EXTENSION IF NOT EXISTS clickhouse_fdw;

-- Create the ClickHouse foreign server
CREATE SERVER clickhouse_server
FOREIGN DATA WRAPPER clickhouse_fdw
OPTIONS (
    dbname 'default',
    host   'localhost',
    port   '8123'
);

-- Create a user mapping
CREATE USER MAPPING FOR CURRENT_USER
SERVER clickhouse_server
OPTIONS (
    user     'default',
    password 'password'
);

-- Verify the server is configured correctly
SELECT srvname, srvtype, srvoptions
FROM pg_foreign_server
WHERE srvname = 'clickhouse_server';
```

## Setting Up ClickHouse Tables

Create the analytical tables in ClickHouse that PostgreSQL will query.

```sql
-- In ClickHouse
CREATE TABLE event_aggregates
(
    dt              Date,
    user_id         UInt64,
    event_type      LowCardinality(String),
    total_events    UInt64,
    total_revenue   Float64,
    session_count   UInt32,
    updated_at      DateTime DEFAULT now()
)
ENGINE = SummingMergeTree((total_events, total_revenue, session_count))
PARTITION BY toYYYYMM(dt)
ORDER BY (user_id, dt, event_type);

-- Populate with sample data
INSERT INTO event_aggregates
SELECT
    today() - number % 90          AS dt,
    number % 10000                  AS user_id,
    ['page_view','click','purchase','signup'][1 + number % 4] AS event_type,
    rand() % 100 + 1               AS total_events,
    round(randCanonical() * 200, 2) AS total_revenue,
    rand() % 10 + 1                AS session_count
FROM numbers(500000);
```

## Creating Foreign Tables in PostgreSQL

Map ClickHouse tables as foreign tables in PostgreSQL.

```sql
-- Create a foreign table mapping to the ClickHouse aggregate table
CREATE FOREIGN TABLE ch_event_aggregates (
    dt            DATE,
    user_id       BIGINT,
    event_type    TEXT,
    total_events  BIGINT,
    total_revenue DOUBLE PRECISION,
    session_count INTEGER,
    updated_at    TIMESTAMP
)
SERVER clickhouse_server
OPTIONS (
    table_name 'event_aggregates'
);

-- Verify the foreign table works
SELECT count(*) FROM ch_event_aggregates;

-- Preview data
SELECT * FROM ch_event_aggregates LIMIT 5;
```

## Importing All Tables Automatically

Use `IMPORT FOREIGN SCHEMA` to import multiple tables at once.

```sql
-- Create a schema for ClickHouse tables
CREATE SCHEMA clickhouse;

-- Import all tables from the ClickHouse default database
IMPORT FOREIGN SCHEMA "default"
FROM SERVER clickhouse_server
INTO clickhouse;

-- List imported tables
SELECT foreign_table_schema, foreign_table_name
FROM information_schema.foreign_tables
WHERE foreign_table_schema = 'clickhouse';
```

## Joining PostgreSQL and ClickHouse Tables

Run cross-database queries that combine transactional and analytical data.

```sql
-- Join PostgreSQL users with ClickHouse event aggregates
SELECT
    u.email,
    u.country,
    u.plan_tier,
    COALESCE(ea.total_events, 0)  AS events_last_30d,
    COALESCE(ea.total_revenue, 0) AS revenue_last_30d
FROM users u
LEFT JOIN ch_event_aggregates ea
       ON ea.user_id = u.id
      AND ea.dt >= CURRENT_DATE - INTERVAL '30 days'
      AND ea.event_type = 'purchase'
WHERE u.created_at >= '2025-01-01'
ORDER BY ea.total_revenue DESC NULLS LAST
LIMIT 100;

-- Aggregate ClickHouse data grouped by PostgreSQL dimensions
SELECT
    u.country,
    u.plan_tier,
    sum(ea.total_events)  AS events,
    sum(ea.total_revenue) AS revenue,
    count(DISTINCT ea.user_id) AS active_users
FROM ch_event_aggregates ea
JOIN users u ON ea.user_id = u.id
WHERE ea.dt >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY u.country, u.plan_tier
ORDER BY revenue DESC;
```

## Using a Materialized View for Performance

Cache frequently accessed ClickHouse data locally in PostgreSQL.

```sql
-- Create a local materialized view from the foreign table
CREATE MATERIALIZED VIEW local_event_summary AS
SELECT
    dt,
    event_type,
    sum(total_events)  AS total_events,
    sum(total_revenue) AS total_revenue,
    count(DISTINCT user_id) AS unique_users
FROM ch_event_aggregates
WHERE dt >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY dt, event_type;

-- Create an index on the materialized view
CREATE INDEX idx_local_event_summary_dt
    ON local_event_summary (dt);

-- Refresh the materialized view periodically
REFRESH MATERIALIZED VIEW CONCURRENTLY local_event_summary;
```

## Writing Back to ClickHouse via FDW

The FDW also supports INSERT operations for writing data back.

```sql
-- Write PostgreSQL data to ClickHouse via FDW
INSERT INTO ch_event_aggregates (dt, user_id, event_type, total_events, total_revenue, session_count)
SELECT
    date_trunc('day', created_at)::DATE AS dt,
    user_id,
    event_type,
    count(*)                            AS total_events,
    sum(amount)                         AS total_revenue,
    count(DISTINCT session_id)          AS session_count
FROM app_events
WHERE created_at::DATE = CURRENT_DATE - 1
GROUP BY 1, 2, 3;
```

## Automating the Refresh

Schedule the materialized view refresh with pg_cron.

```sql
-- Install pg_cron
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Refresh the summary every 15 minutes
SELECT cron.schedule(
    'refresh-event-summary',
    '*/15 * * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY local_event_summary'
);

-- Verify the scheduled job
SELECT * FROM cron.job WHERE jobname = 'refresh-event-summary';
```

## Troubleshooting Common Issues

Diagnose and fix common FDW problems.

```sql
-- Check connection options
SELECT *
FROM pg_foreign_server
WHERE srvname = 'clickhouse_server';

-- Test with a simple query and enable debug output
SET client_min_messages = DEBUG;
SELECT count(*) FROM ch_event_aggregates;
RESET client_min_messages;

-- Check for pushdown of WHERE clauses
EXPLAIN (VERBOSE, ANALYZE)
SELECT * FROM ch_event_aggregates
WHERE dt = '2026-03-31'
  AND event_type = 'purchase';
```

## Summary

The `clickhouse_fdw` extension bridges PostgreSQL and ClickHouse by exposing ClickHouse tables as foreign tables in PostgreSQL. This enables cross-database joins in standard SQL, letting you enrich transactional data with ClickHouse analytics without building a separate data pipeline. Use `IMPORT FOREIGN SCHEMA` to quickly import all tables, create local materialized views for hot data to avoid per-query network round-trips, and use pg_cron to keep materialized views fresh.
