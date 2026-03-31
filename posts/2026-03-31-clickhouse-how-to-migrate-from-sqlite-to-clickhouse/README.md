# How to Migrate from SQLite to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQLite, Migration, Analytics, Database, ETL

Description: Migrate data from SQLite to ClickHouse when your analytical dataset outgrows SQLite's single-file limitations and you need faster query performance.

---

SQLite is excellent for embedded and small-scale applications, but its single-writer model and lack of columnar storage make it unsuitable for analytical workloads as data grows. ClickHouse handles the same SQL queries on billions of rows in milliseconds.

## When to Migrate from SQLite to ClickHouse

- Query times exceed seconds on analytical queries
- Dataset exceeds a few GB
- You need concurrent analytical queries from multiple clients
- You want aggregation performance for dashboards or reports

## Step 1 - Export Data from SQLite

Use the SQLite CLI to export to CSV:

```bash
sqlite3 myapp.db -header -csv "SELECT * FROM events;" > events.csv
```

For multiple tables:

```bash
sqlite3 myapp.db <<'EOF'
.mode csv
.headers on
.output /tmp/events.csv
SELECT * FROM events;
.output /tmp/users.csv
SELECT * FROM users;
.output /tmp/page_views.csv
SELECT * FROM page_views;
EOF
```

## Step 2 - Create Tables in ClickHouse

Translate SQLite schema to ClickHouse DDL:

```sql
-- SQLite schema
-- CREATE TABLE events (
--     id INTEGER PRIMARY KEY,
--     user_id TEXT,
--     event_type TEXT,
--     created_at TEXT
-- );

-- ClickHouse equivalent
CREATE TABLE analytics.events
(
    id           UInt64,
    user_id      String,
    event_type   LowCardinality(String),
    created_at   DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id);
```

## Step 3 - Load CSV into ClickHouse

```bash
clickhouse-client \
  --query "INSERT INTO analytics.events FORMAT CSVWithNames" \
  < /tmp/events.csv
```

Or use the HTTP interface:

```bash
curl -X POST 'http://localhost:8123/?query=INSERT+INTO+analytics.events+FORMAT+CSVWithNames' \
  --data-binary @/tmp/events.csv
```

## Step 4 - Handle SQLite Date Strings

SQLite stores dates as ISO 8601 strings. Parse them in ClickHouse:

```sql
INSERT INTO analytics.events
SELECT
    id,
    user_id,
    event_type,
    parseDateTimeBestEffort(created_at) AS created_at
FROM input('id UInt64, user_id String, event_type String, created_at String')
FORMAT CSVWithNames;
```

## Step 5 - Compare Query Results

Run equivalent queries on both to validate the migration:

SQLite:
```sql
SELECT event_type, COUNT(*) AS cnt
FROM events
GROUP BY event_type
ORDER BY cnt DESC;
```

ClickHouse:
```sql
SELECT event_type, count() AS cnt
FROM analytics.events
GROUP BY event_type
ORDER BY cnt DESC;
```

## Step 6 - Redirect Application Queries

Update your application connection string and replace SQLite-specific syntax:

```python
# Before (SQLite)
conn = sqlite3.connect('myapp.db')

# After (ClickHouse)
import clickhouse_connect
client = clickhouse_connect.get_client(host='localhost', port=8123)
```

## Summary

Migrating from SQLite to ClickHouse is straightforward via CSV export and import. The SQL syntax is similar enough that most queries need only minor adjustments. Once migrated, analytical queries that took seconds in SQLite typically complete in milliseconds on ClickHouse, even as your dataset grows to billions of rows.
