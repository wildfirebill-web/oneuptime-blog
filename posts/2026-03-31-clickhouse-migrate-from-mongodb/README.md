# How to Migrate from MongoDB to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MongoDB, Migration, Analytics, NoSQL, OLAP

Description: Migrate analytical workloads from MongoDB to ClickHouse using mongoexport, the MongoDB table engine, or a CDC-based approach for ongoing sync.

---

MongoDB is a great document store for operational data, but aggregation pipelines over large collections are slow compared to ClickHouse's columnar engine. Moving analytical queries to ClickHouse frees up MongoDB for its intended purpose.

## Option 1 - ClickHouse MongoDB Table Engine

ClickHouse can query MongoDB directly via the MongoDB table engine:

```sql
CREATE TABLE mongo_events
ENGINE = MongoDB(
    'mongodb://user:password@mongo-host:27017/analytics',
    'events',
    'event_id UInt64, user_id UInt64, event_type String, created_at DateTime, revenue Float64'
);
```

Copy to a MergeTree table:

```sql
INSERT INTO events SELECT * FROM mongo_events;
```

## Option 2 - Export with mongoexport

Export as JSON Lines:

```bash
mongoexport \
  --host mongo-host \
  --db analytics \
  --collection events \
  --fields event_id,user_id,event_type,created_at,revenue \
  --out events.jsonl
```

Load into ClickHouse:

```bash
clickhouse-client \
  --query "INSERT INTO events FORMAT JSONEachRow" \
  < events.jsonl
```

Export as CSV:

```bash
mongoexport \
  --host mongo-host \
  --db analytics \
  --collection events \
  --type csv \
  --fields event_id,user_id,event_type,created_at,revenue \
  --out events.csv
```

## Handling Nested Documents

MongoDB documents often have nested objects. Flatten them during export using an aggregation pipeline:

```bash
mongoexport \
  --host mongo-host \
  --db analytics \
  --collection events \
  --query '{"$project": {"event_id": 1, "user_id": 1, "browser": "$properties.browser"}}' \
  --out events_flat.jsonl
```

In ClickHouse, store properties as a `Map` type for flexible access:

```sql
CREATE TABLE events (
    event_id   UInt64,
    user_id    UInt64,
    event_type LowCardinality(String),
    created_at DateTime,
    properties Map(String, String)
) ENGINE = MergeTree()
ORDER BY (created_at, user_id);
```

## Schema Translation

MongoDB stores documents without a fixed schema. Extract the schema from a sample:

```javascript
db.events.findOne()
// {
//   "_id": ObjectId("..."),
//   "user_id": 12345,
//   "event_type": "purchase",
//   "created_at": ISODate("2024-01-15T10:30:00Z"),
//   "revenue": 49.99
// }
```

ClickHouse DDL:

```sql
CREATE TABLE events (
    event_id   UUID,
    user_id    UInt64,
    event_type LowCardinality(String),
    created_at DateTime,
    revenue    Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id);
```

Note: MongoDB `_id` ObjectId does not map directly to ClickHouse types. Use `UUID` with `generateUUIDv4()` or drop it if not needed.

## Query Translation

MongoDB aggregation pipeline:

```javascript
db.events.aggregate([
  { $match: { created_at: { $gte: new Date("2024-01-01") } } },
  { $group: { _id: "$event_type", total: { $sum: "$revenue" }, count: { $sum: 1 } } },
  { $sort: { total: -1 } },
  { $limit: 10 }
])
```

ClickHouse SQL:

```sql
SELECT
    event_type,
    sum(revenue) AS total,
    count() AS cnt
FROM events
WHERE created_at >= '2024-01-01'
GROUP BY event_type
ORDER BY total DESC
LIMIT 10;
```

## Summary

Migrating from MongoDB to ClickHouse replaces complex aggregation pipelines with standard SQL. The MongoDB table engine enables a gradual migration, while `mongoexport` handles one-time bulk moves. Flatten nested documents during export to take full advantage of ClickHouse's columnar structure.
