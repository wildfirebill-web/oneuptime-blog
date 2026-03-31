# How to Add Columns to a ClickHouse Table

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, DDL, ALTER TABLE, Schema Migration

Description: Learn how to add columns to a ClickHouse table using ALTER TABLE ADD COLUMN, including AFTER, FIRST, default values, and materialized expressions.

---

Adding columns to a ClickHouse table is a metadata-only operation for MergeTree-family tables - it does not rewrite existing data parts immediately. The new column is added to the schema and existing rows return a default value until parts are merged or the column is materialized. This makes column additions fast even on large tables.

## Basic ADD COLUMN Syntax

The simplest form adds a column to the end of the column list:

```sql
ALTER TABLE events
    ADD COLUMN browser String;
```

To add multiple columns in a single statement, separate each clause with a comma:

```sql
ALTER TABLE events
    ADD COLUMN browser String,
    ADD COLUMN os String,
    ADD COLUMN device_type LowCardinality(String);
```

## Controlling Column Position

### AFTER Clause

Use `AFTER` to insert the new column immediately after an existing one:

```sql
ALTER TABLE events
    ADD COLUMN response_time_ms UInt32 AFTER status_code;
```

### FIRST Clause

Use `FIRST` to place the column at the very beginning of the schema:

```sql
ALTER TABLE events
    ADD COLUMN tenant_id UUID FIRST;
```

## Default Values

ClickHouse supports three kinds of default expressions for new columns.

### DEFAULT - Stored on Insert

A `DEFAULT` expression is evaluated at insert time if no value is provided. Existing rows return the expression result when read until the part is merged:

```sql
ALTER TABLE events
    ADD COLUMN environment LowCardinality(String) DEFAULT 'production';
```

### MATERIALIZED - Computed and Stored

A `MATERIALIZED` column is always computed from other columns and stored on disk. It cannot be inserted into directly:

```sql
ALTER TABLE events
    ADD COLUMN event_date Date MATERIALIZED toDate(created_at);
```

After adding a materialized column, trigger backfill so existing parts include the value:

```sql
ALTER TABLE events
    MATERIALIZE COLUMN event_date;
```

### ALIAS - Computed on Read, Not Stored

An `ALIAS` column is computed at query time and never stored:

```sql
ALTER TABLE events
    ADD COLUMN full_url String ALIAS concat(scheme, '://', host, path);
```

## Complete Example

The following example builds a table and then extends its schema with several column additions:

```sql
-- Create base table
CREATE TABLE http_logs
(
    created_at DateTime,
    host       String,
    path       String,
    status     UInt16,
    bytes      UInt64
)
ENGINE = MergeTree
ORDER BY (host, created_at);

-- Add a column after an existing one
ALTER TABLE http_logs
    ADD COLUMN referer String DEFAULT '' AFTER path;

-- Add a materialized date column for partitioning queries
ALTER TABLE http_logs
    ADD COLUMN log_date Date MATERIALIZED toDate(created_at);

-- Backfill the materialized column in existing parts
ALTER TABLE http_logs
    MATERIALIZE COLUMN log_date;

-- Verify the new schema
DESCRIBE TABLE http_logs;
```

## ON CLUSTER for Distributed Setups

In a ClickHouse cluster, use `ON CLUSTER` to propagate the DDL to every replica and shard automatically:

```sql
ALTER TABLE http_logs ON CLUSTER '{cluster}'
    ADD COLUMN region LowCardinality(String) DEFAULT 'us-east-1';
```

The `'{cluster}'` macro is resolved by each node from its `config.xml`. Without `ON CLUSTER` you would need to run the statement on every node manually.

## Checking Column Existence Before Adding

To avoid errors when running migrations idempotently, check whether a column already exists before adding it:

```sql
SELECT name
FROM system.columns
WHERE database = 'default'
  AND table = 'http_logs'
  AND name = 'region';
```

If the query returns no rows, it is safe to run the `ADD COLUMN` statement.

## Summary

`ALTER TABLE ADD COLUMN` in ClickHouse is a lightweight metadata operation that supports precise positioning with `AFTER` and `FIRST`, and flexible defaults via `DEFAULT`, `MATERIALIZED`, and `ALIAS`. For distributed clusters, always include `ON CLUSTER` to keep all nodes in sync. Materialized columns require a separate `MATERIALIZE COLUMN` step to backfill existing data parts.
