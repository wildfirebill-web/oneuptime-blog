# How to Sync Redis Data to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Redis, Data Sync, ETL, Analytics

Description: Sync Redis data to ClickHouse for historical analytics and time-series analysis using batch export or streaming pub/sub pipelines.

---

## Overview

Redis is an in-memory data store optimized for low-latency reads and writes, but it is not designed for analytical queries over large datasets. Syncing Redis data to ClickHouse lets you run complex SQL aggregations and retain historical data beyond Redis's memory limits.

## Common Use Cases

```text
- Session metrics stored in Redis hashes -> analyze user behavior in ClickHouse
- Redis counters (page views, clicks) -> time-series analysis in ClickHouse
- Redis Streams -> real-time event ingestion via ClickHouse Kafka engine
- Leaderboard snapshots -> historical rank tracking
```

## Approach 1: Batch Export with redis-cli

Dump Redis hash data to a CSV file and load it into ClickHouse:

```bash
redis-cli --no-auth-warning \
  HGETALL session:stats:20260331 | \
  paste - - | \
  awk '{print "2026-03-31\t" $1 "\t" $2}' > /tmp/sessions.tsv
```

Load into ClickHouse:

```bash
clickhouse-client --query="INSERT INTO session_stats FORMAT TSV" < /tmp/sessions.tsv
```

Create the target table first:

```sql
CREATE TABLE session_stats (
    stat_date   Date,
    metric      LowCardinality(String),
    value       Float64
) ENGINE = MergeTree()
ORDER BY (metric, stat_date);
```

## Approach 2: Redis Streams to Kafka

Publish events to Redis Streams, then bridge to Kafka with a connector, and consume in ClickHouse:

```bash
# Produce to Redis Stream
redis-cli XADD events:clicks '*' \
  user_id 12345 \
  page /home \
  ts 1743375600
```

Use Kafka Connect's Redis source connector to bridge to Kafka, then consume in ClickHouse with the Kafka engine.

## Approach 3: Python Sync Script

Write a Python script that reads Redis keys and writes to ClickHouse:

```python
import redis
import clickhouse_connect
from datetime import datetime

r = redis.Redis(host='localhost', port=6379)
ch = clickhouse_connect.get_client(host='localhost')

today = datetime.today().strftime('%Y-%m-%d')
keys = r.keys('counter:*')

rows = []
for key in keys:
    metric = key.decode().replace('counter:', '')
    value = float(r.get(key) or 0)
    rows.append([today, metric, value])

ch.insert('daily_counters', rows,
    column_names=['stat_date', 'metric', 'value'])
print(f'Synced {len(rows)} counters')
```

Target table:

```sql
CREATE TABLE daily_counters (
    stat_date   Date,
    metric      LowCardinality(String),
    value       Float64
) ENGINE = SummingMergeTree(value)
ORDER BY (metric, stat_date);
```

## Approach 4: Redis Dictionary in ClickHouse

For real-time lookups of Redis values in ClickHouse queries, use a Redis dictionary:

```sql
CREATE DICTIONARY redis_feature_flags (
    flag_name  String,
    enabled    UInt8
)
PRIMARY KEY flag_name
SOURCE(REDIS(
    HOST 'redis-host'
    PORT 6379
    DB_INDEX 0
    STORAGE_TYPE simple
))
LAYOUT(FLAT())
LIFETIME(60);
```

Query it inline:

```sql
SELECT event_id
FROM events
WHERE dictGet('redis_feature_flags', 'enabled', toString(feature_name)) = 1;
```

## Summary

Redis and ClickHouse serve different roles: Redis for fast operational reads and writes, ClickHouse for analytical queries over historical data. Sync Redis counters and metrics to ClickHouse via scheduled batch scripts or streaming pipelines to build dashboards and trend reports on your Redis data.
