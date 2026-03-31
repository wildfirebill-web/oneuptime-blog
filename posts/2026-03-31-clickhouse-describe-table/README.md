# How to Use DESCRIBE TABLE in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, DESCRIBE, Schema, Metadata

Description: Learn how to use DESCRIBE TABLE in ClickHouse to inspect column names, types, defaults, and comments for any table or view.

---

`DESCRIBE TABLE` in ClickHouse returns metadata about the columns in a table, view, or other table-like object. It is one of the quickest ways to understand a table's schema without reading the full CREATE statement, and it is especially useful when exploring an unfamiliar database or verifying that a migration applied the expected column structure.

## Basic DESCRIBE TABLE Syntax

```sql
DESCRIBE TABLE events;
```

Sample output:

```text
name          | type        | default_type | default_expression | comment | codec_expression | ttl_expression
event_id      | UUID        |              |                    |         |                  |
user_id       | UInt64      |              |                    |         |                  |
event_name    | String      |              |                    |         |                  |
event_date    | Date        |              |                    |         |                  |
value         | Float64     |              |                    |         |                  |
```

Each row represents one column. The columns returned are:
- `name` - column name
- `type` - data type
- `default_type` - one of `DEFAULT`, `MATERIALIZED`, `ALIAS`, or empty
- `default_expression` - the expression if a default is defined
- `comment` - column-level comment
- `codec_expression` - compression codec if specified
- `ttl_expression` - TTL rule if set on the column

## Describing a Table with Defaults and Comments

```sql
CREATE TABLE orders
(
    order_id    UUID         DEFAULT generateUUIDv4() COMMENT 'Auto-generated order identifier',
    user_id     UInt64       COMMENT 'References users.user_id',
    total       Float64      DEFAULT 0.0,
    status      LowCardinality(String) DEFAULT 'pending',
    created_at  DateTime     DEFAULT now()
)
ENGINE = MergeTree()
ORDER BY (user_id, created_at);

DESCRIBE TABLE orders;
```

```text
name       | type                    | default_type | default_expression    | comment
order_id   | UUID                    | DEFAULT      | generateUUIDv4()      | Auto-generated order identifier
user_id    | UInt64                  |              |                       | References users.user_id
total      | Float64                 | DEFAULT      | 0.0                   |
status     | LowCardinality(String)  | DEFAULT      | 'pending'             |
created_at | DateTime                | DEFAULT      | now()                 |
```

Default expressions and comments are shown inline, which saves you from querying `system.columns` for routine schema inspection.

## Describing a View

DESCRIBE TABLE works on views as well as physical tables:

```sql
CREATE VIEW daily_revenue AS
SELECT
    toDate(created_at) AS day,
    sum(total)         AS revenue
FROM orders
GROUP BY day;

DESCRIBE TABLE daily_revenue;
```

```text
name    | type    | default_type | default_expression | comment
day     | Date    |              |                    |
revenue | Float64 |              |                    |
```

Views return the output column names and inferred types without exposing the underlying SELECT.

## Describing MATERIALIZED and ALIAS Columns

```sql
CREATE TABLE sensor_readings
(
    sensor_id    UInt32,
    recorded_at  DateTime,
    celsius      Float32,
    fahrenheit   Float32 MATERIALIZED celsius * 9 / 5 + 32,
    date_only    Date    ALIAS toDate(recorded_at)
)
ENGINE = MergeTree()
ORDER BY (sensor_id, recorded_at);

DESCRIBE TABLE sensor_readings;
```

```text
name         | type     | default_type | default_expression
sensor_id    | UInt32   |              |
recorded_at  | DateTime |              |
celsius      | Float32  |              |
fahrenheit   | Float32  | MATERIALIZED | celsius * 9 / 5 + 32
date_only    | Date     | ALIAS        | toDate(recorded_at)
```

`MATERIALIZED` columns are computed at insert time and stored on disk. `ALIAS` columns are computed on read and not stored. Both appear in DESCRIBE TABLE with their expressions.

## Filtering DESCRIBE Output with LIKE

You can pipe DESCRIBE results into a query to filter columns:

```sql
SELECT name, type, comment
FROM (DESCRIBE TABLE orders)
WHERE name LIKE '%id%';
```

```text
name     | type   | comment
order_id | UUID   | Auto-generated order identifier
user_id  | UInt64 | References users.user_id
```

This is useful when a table has many columns and you only need to locate columns by naming convention.

## Comparing DESCRIBE TABLE vs system.columns

For programmatic access, `system.columns` provides the same data with additional fields:

```sql
-- DESCRIBE TABLE: quick interactive use
DESCRIBE TABLE events;

-- system.columns: full metadata, filterable, joinable
SELECT name, type, default_kind, default_expression, comment
FROM system.columns
WHERE database = 'default' AND table = 'events'
ORDER BY position;
```

Use `DESCRIBE TABLE` for quick interactive inspection and `system.columns` when you need to query column metadata programmatically or join it with other system tables.

## Summary

`DESCRIBE TABLE` is the fastest way to inspect a table's column names, types, default expressions, and comments in ClickHouse. It works on physical tables, views, and Materialized Views alike. For deeper metadata queries - such as filtering columns by pattern or joining schema information with table statistics - `system.columns` provides the same data in a fully queryable form.
