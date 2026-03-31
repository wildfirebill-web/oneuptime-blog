# How to Use SHOW CREATE TABLE in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, SHOW, DDL, Schema

Description: Learn how to use SHOW CREATE TABLE in ClickHouse to retrieve the full DDL for tables, views, dictionaries, and databases.

---

`SHOW CREATE TABLE` in ClickHouse returns the complete DDL statement that would recreate a table exactly as it exists today - including the engine, ORDER BY keys, partition expressions, codec settings, TTL rules, and column comments. It is an essential command for schema migrations, backups, and auditing the exact configuration of production tables.

## Basic SHOW CREATE TABLE Syntax

```sql
SHOW CREATE TABLE events;
```

Sample output:

```sql
CREATE TABLE default.events
(
    `event_id`   UUID,
    `user_id`    UInt64,
    `event_name` String,
    `event_date` Date,
    `value`      Float64
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (user_id, event_date)
SETTINGS index_granularity = 8192
```

The output is a single `statement` column containing the complete, copy-pasteable DDL.

## Viewing a Table with Advanced Settings

```sql
SHOW CREATE TABLE orders;
```

```sql
CREATE TABLE default.orders
(
    `order_id`   UUID              DEFAULT generateUUIDv4() COMMENT 'Auto-generated order ID',
    `user_id`    UInt64,
    `total`      Float64           DEFAULT 0.0,
    `status`     LowCardinality(String) DEFAULT 'pending',
    `created_at` DateTime          DEFAULT now(),
    `amount_usd` Float64           MATERIALIZED total * 1.0
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (user_id, created_at)
TTL created_at + INTERVAL 2 YEAR DELETE
SETTINGS index_granularity = 8192
```

All defaults, MATERIALIZED expressions, COMMENT clauses, TTL rules, and engine settings are included verbatim.

## SHOW CREATE VIEW

Views return their SELECT definition:

```sql
CREATE VIEW daily_revenue AS
SELECT toDate(created_at) AS day, sum(total) AS revenue
FROM orders
GROUP BY day;

SHOW CREATE VIEW daily_revenue;
```

```sql
CREATE VIEW default.daily_revenue
(
    `day`     Date,
    `revenue` Float64
) AS
SELECT
    toDate(created_at) AS day,
    sum(total) AS revenue
FROM orders
GROUP BY day
```

## SHOW CREATE DICTIONARY

External dictionaries store their source, layout, and lifetime configuration in the DDL:

```sql
SHOW CREATE DICTIONARY region_names;
```

```sql
CREATE DICTIONARY default.region_names
(
    `region_id`   UInt32,
    `region_name` String DEFAULT ''
)
PRIMARY KEY region_id
SOURCE(CLICKHOUSE(HOST 'localhost' PORT 9000 USER 'default' TABLE 'regions' DB 'default'))
LIFETIME(MIN 300 MAX 360)
LAYOUT(HASHED())
```

This is useful for auditing dictionary refresh intervals and source configurations.

## SHOW CREATE DATABASE

```sql
SHOW CREATE DATABASE analytics;
```

```sql
CREATE DATABASE analytics
ENGINE = Atomic
```

For databases using engines like `Replicated` or `MySQL`, the engine parameters are included in the output.

## Using SHOW CREATE for Migration and Backup

A common pattern is to export DDL for all tables in a database:

```bash
# Export DDL for all tables in a database using clickhouse-client
clickhouse-client --query "
SELECT 'SHOW CREATE TABLE ' || name || ';'
FROM system.tables
WHERE database = 'analytics'
" | clickhouse-client --multiquery > analytics_schema_backup.sql
```

Or use the simpler approach of querying `system.tables` directly:

```sql
-- Get CREATE TABLE statements for all MergeTree tables in a database
SELECT create_table_query
FROM system.tables
WHERE database = 'analytics'
  AND engine LIKE '%MergeTree%';
```

`system.tables` exposes the `create_table_query` column which is equivalent to `SHOW CREATE TABLE` output for every table.

## Comparing SHOW CREATE TABLE vs system.tables

```sql
-- SHOW CREATE TABLE: interactive, returns formatted DDL
SHOW CREATE TABLE events;

-- system.tables: programmatic, can filter and aggregate
SELECT name, create_table_query
FROM system.tables
WHERE database = currentDatabase()
ORDER BY name;
```

Use `SHOW CREATE TABLE` for quick interactive inspection of a single table and `system.tables` when you need to iterate over multiple tables or include DDL in automated scripts.

## Summary

`SHOW CREATE TABLE` retrieves the exact DDL needed to recreate a ClickHouse table, view, dictionary, or database, including all engine parameters, partition keys, TTL rules, and column-level settings. It is the most reliable way to capture a schema snapshot for migration or backup purposes. For bulk operations across many tables, use the equivalent `create_table_query` column in `system.tables` to query and export DDL programmatically.
