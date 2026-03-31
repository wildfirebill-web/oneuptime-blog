# How to Build Your First ClickHouse Table in 5 Minutes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Beginner, Getting Started, MergeTree, Table, Tutorial

Description: Create your first ClickHouse table in 5 minutes - choose the right engine, define columns, set the primary key, and insert your first rows.

---

## Prerequisites

Make sure ClickHouse is running. If using Docker:

```bash
docker run -d \
  --name clickhouse-server \
  -p 9000:9000 -p 8123:8123 \
  clickhouse/clickhouse-server
```

Connect to it:

```bash
docker exec -it clickhouse-server clickhouse-client
```

## Step 1 - Choose the Right Engine

For almost all use cases, use `MergeTree`. It supports efficient inserts, sorting, compression, and TTL. Other engines are specializations of it:

| Engine | Use Case |
|--------|---------|
| `MergeTree` | General purpose |
| `ReplicatedMergeTree` | Distributed, replicated |
| `SummingMergeTree` | Pre-aggregated sums |
| `ReplacingMergeTree` | Upsert/deduplication |

## Step 2 - Define Your Table

Create a simple events table:

```sql
CREATE TABLE events
(
    event_id    UUID DEFAULT generateUUIDv4(),
    event_type  LowCardinality(String),
    user_id     UInt64,
    session_id  String,
    ts          DateTime,
    amount      Nullable(Float64),
    country     LowCardinality(String)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (event_type, user_id, ts);
```

Key decisions:
- `PARTITION BY toYYYYMM(ts)` - splits data into monthly partitions for easy cleanup
- `ORDER BY (event_type, user_id, ts)` - the primary sort key; queries filtering by these columns are fast

## Step 3 - Insert Data

```sql
INSERT INTO events (event_type, user_id, session_id, ts, amount, country) VALUES
('purchase', 42, 'sess_001', '2026-03-01 10:00:00', 99.99, 'US'),
('page_view', 42, 'sess_001', '2026-03-01 10:01:00', NULL, 'US'),
('purchase', 99, 'sess_002', '2026-03-01 11:00:00', 49.50, 'DE');
```

Or use JSONEachRow format:

```bash
echo '{"event_type":"purchase","user_id":42,"session_id":"s1","ts":"2026-03-01 12:00:00","amount":19.99,"country":"US"}' | \
  clickhouse-client --query="INSERT INTO events FORMAT JSONEachRow"
```

## Step 4 - Run Your First Query

Count events by type:

```sql
SELECT
    event_type,
    count()    AS total,
    sum(amount) AS total_amount
FROM events
GROUP BY event_type
ORDER BY total DESC;
```

Check table size:

```sql
SELECT
    formatReadableSize(sum(bytes_on_disk)) AS disk_size,
    sum(rows) AS total_rows,
    count() AS parts
FROM system.parts
WHERE table = 'events' AND active;
```

## Step 5 - Add TTL for Automatic Data Expiry

Delete data older than 90 days automatically:

```sql
ALTER TABLE events
MODIFY TTL ts + INTERVAL 90 DAY;
```

## Summary

Build your first ClickHouse table by choosing MergeTree as the engine, defining columns with appropriate types (LowCardinality for repeated strings, Nullable only when needed), setting a partition key for time-based cleanup, and an ORDER BY key aligned with your most common query filters. Insert data in batches via VALUES or JSONEachRow and query immediately.
