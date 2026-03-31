# How to Migrate from SQLite to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQLite, Migration, Analytics, Database

Description: Migrate your data from SQLite to ClickHouse for scalable analytics, covering schema translation, CSV export, and direct SQLite table engine reading.

---

SQLite is great for embedded applications and small datasets. When your data grows to millions of rows and analytical queries become slow, ClickHouse is a natural upgrade. This guide covers two migration paths: CSV export and the built-in SQLite table engine.

## Option 1 - Use the ClickHouse SQLite Table Engine

ClickHouse can read directly from an SQLite file using the SQLite table engine:

```sql
CREATE TABLE sqlite_source
ENGINE = SQLite('/path/to/data.sqlite', 'events');
```

Then insert into a proper MergeTree table:

```sql
CREATE TABLE events (
    id         UInt64,
    user_id    UInt64,
    event_type LowCardinality(String),
    created_at DateTime,
    value      Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id);

INSERT INTO events SELECT * FROM sqlite_source;
```

This is the simplest approach for files accessible on the ClickHouse server.

## Option 2 - CSV Export

Export from SQLite using the CLI:

```bash
sqlite3 -header -csv data.sqlite "SELECT * FROM events;" > events.csv
```

For large tables, export in date batches:

```bash
sqlite3 -header -csv data.sqlite \
  "SELECT * FROM events WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01';" \
  > events_2024_01.csv
```

## Translating the Schema

SQLite DDL:

```sql
CREATE TABLE events (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id    INTEGER,
    event_type TEXT,
    created_at TEXT,
    value      REAL
);
```

ClickHouse equivalent:

```sql
CREATE TABLE events (
    id         UInt64,
    user_id    UInt64,
    event_type LowCardinality(String),
    created_at DateTime,
    value      Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id);
```

Note: SQLite stores dates as TEXT. Cast during import:

```sql
INSERT INTO events
SELECT
    id,
    user_id,
    event_type,
    parseDateTimeBestEffort(created_at),
    value
FROM sqlite_source;
```

## Loading CSV into ClickHouse

```bash
clickhouse-client \
  --query "INSERT INTO events FORMAT CSVWithNames" \
  < events.csv
```

If dates are in SQLite TEXT format (`YYYY-MM-DD HH:MM:SS`), ClickHouse parses them automatically with `CSVWithNames`.

## Query Translation

SQLite uses `strftime` for date operations. ClickHouse has native date functions:

SQLite:

```sql
SELECT strftime('%Y-%m', created_at) AS month, COUNT(*) AS cnt
FROM events
GROUP BY month
ORDER BY month;
```

ClickHouse:

```sql
SELECT toYYYYMM(created_at) AS month, count() AS cnt
FROM events
GROUP BY month
ORDER BY month;
```

## Validating the Migration

```sql
-- Compare counts (run the same in sqlite3 and ClickHouse)
SELECT count() FROM events;

-- Check min/max dates
SELECT min(created_at), max(created_at) FROM events;
```

## Summary

Migrating from SQLite to ClickHouse is one of the easiest database migrations available. The SQLite table engine lets you read the file directly without any intermediate steps, while CSV export gives you a portable option for remote migrations.
