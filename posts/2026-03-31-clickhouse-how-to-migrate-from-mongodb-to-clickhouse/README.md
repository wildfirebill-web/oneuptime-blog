# How to Migrate from MongoDB to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MongoDB, Migration, Analytics, Document Store, ETL

Description: Migrate analytical workloads from MongoDB to ClickHouse to replace slow aggregation pipelines with fast SQL queries on flattened, columnar data.

---

MongoDB's document model and aggregation pipeline work well for operational queries, but they struggle with the kind of large-scale GROUP BY and time-series analytics that ClickHouse handles natively. Migrating analytical data to ClickHouse is a common way to unblock reporting and dashboards.

## When to Keep MongoDB vs. Migrate

Keep MongoDB for:
- Flexible document storage with nested objects
- CRUD operations on individual documents
- Real-time operational data

Migrate to ClickHouse for:
- Aggregation queries over millions of documents
- Time-series dashboards
- Cross-collection analytical joins

## Step 1 - Export Data from MongoDB

Use `mongoexport` to export to JSON Lines:

```bash
mongoexport \
  --host localhost \
  --db analytics \
  --collection events \
  --type json \
  --out /tmp/events.jsonl
```

For large collections, filter by date range:

```bash
mongoexport \
  --host localhost \
  --db analytics \
  --collection events \
  --query '{"created_at": {"$gte": {"$date": "2024-01-01T00:00:00Z"}}}' \
  --out /tmp/events_2024.jsonl
```

## Step 2 - Flatten Nested Documents

MongoDB documents may have nested fields. Flatten them using `jq` before loading:

```bash
jq -c '{
  event_id: ._id."$oid",
  user_id: .userId,
  event_type: .type,
  page: .properties.page,
  duration_ms: (.properties.duration // 0),
  created_at: .created_at."$date"
}' /tmp/events.jsonl > /tmp/events_flat.jsonl
```

## Step 3 - Create the ClickHouse Table

```sql
CREATE TABLE analytics.events
(
    event_id    String,
    user_id     String,
    event_type  LowCardinality(String),
    page        LowCardinality(String),
    duration_ms UInt32 DEFAULT 0,
    created_at  DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id, event_type);
```

## Step 4 - Load the Flattened JSON

```bash
clickhouse-client \
  --query "INSERT INTO analytics.events FORMAT JSONEachRow" \
  < /tmp/events_flat.jsonl
```

## Step 5 - Translate MongoDB Aggregation to ClickHouse SQL

MongoDB aggregation pipeline:

```javascript
db.events.aggregate([
  { $match: { created_at: { $gte: new Date("2024-01-01") } } },
  { $group: {
    _id: { type: "$type", day: { $dateToString: { format: "%Y-%m-%d", date: "$created_at" } } },
    count: { $sum: 1 },
    unique_users: { $addToSet: "$userId" }
  }},
  { $sort: { "_id.day": 1 } }
])
```

ClickHouse equivalent:

```sql
SELECT
    event_type,
    toDate(created_at) AS day,
    count() AS cnt,
    uniq(user_id) AS unique_users
FROM analytics.events
WHERE created_at >= '2024-01-01'
GROUP BY event_type, day
ORDER BY day;
```

## Step 6 - Set Up Ongoing Sync with Change Streams

Use Debezium MongoDB connector to stream changes to Kafka, then load into ClickHouse via the Kafka engine for real-time replication.

## Summary

Migrating analytical data from MongoDB to ClickHouse involves flattening nested documents, loading them as JSON Lines, and rewriting aggregation pipelines as SQL. The result is typically 10-100x faster query performance for analytical workloads, with a simpler query syntax and much lower storage overhead.
