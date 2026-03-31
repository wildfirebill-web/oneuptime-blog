# ClickHouse for MongoDB Developers - Key Differences

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MongoDB, Migration, Analytics, SQL

Description: A practical guide for MongoDB developers learning ClickHouse, covering key differences in data model, query language, and use case fit.

---

MongoDB and ClickHouse have very different design philosophies. MongoDB is a document store designed for flexible schema OLTP workloads. ClickHouse is a columnar database built for analytical queries over massive datasets. When your MongoDB analytics queries are slow, ClickHouse is often the answer.

## Document Model vs. Columnar Model

MongoDB stores JSON documents with nested fields. ClickHouse uses a strict table schema with typed columns. You need to flatten nested documents when migrating:

```javascript
// MongoDB document
{
  "_id": "abc123",
  "user": { "id": 42, "country": "US" },
  "event": "purchase",
  "amount": 99.99,
  "timestamp": ISODate("2024-01-01")
}
```

```sql
-- ClickHouse equivalent: flattened schema
CREATE TABLE events (
    event_id String,
    user_id UInt32,
    user_country LowCardinality(String),
    event String,
    amount Decimal(10, 2),
    event_time DateTime
) ENGINE = MergeTree()
ORDER BY (user_id, event_time);
```

## Aggregation Pipeline vs. SQL

MongoDB uses an aggregation pipeline with stages. ClickHouse uses standard SQL with powerful aggregate functions:

```javascript
// MongoDB aggregation
db.events.aggregate([
  { $match: { event: "purchase" } },
  { $group: { _id: "$user.country", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } }
])
```

```sql
-- ClickHouse SQL equivalent
SELECT
    user_country,
    sum(amount) AS total
FROM events
WHERE event = 'purchase'
GROUP BY user_country
ORDER BY total DESC;
```

## Schema Flexibility

MongoDB is schema-less by default. ClickHouse requires a defined schema, but you can handle semi-structured data with JSON columns:

```sql
CREATE TABLE events (
    event_time DateTime,
    event_type String,
    properties JSON
) ENGINE = MergeTree()
ORDER BY event_time;

-- Query JSON fields
SELECT
    event_type,
    JSONExtractString(properties, 'page_url') AS page
FROM events
LIMIT 10;
```

## No Document-Level Locks

MongoDB has document-level locking. ClickHouse has no row-level locking - writes are block-level and appended to new data parts. This makes ClickHouse extremely fast for bulk inserts but means you cannot update individual documents cheaply.

## Indexing Approach

MongoDB uses B-tree secondary indexes on any field. ClickHouse's primary index is the sorting key, and secondary indexing uses skip indexes:

```sql
-- ClickHouse skip index (probabilistic, not exact)
ALTER TABLE events
ADD INDEX idx_event_type event_type TYPE bloom_filter GRANULARITY 4;

-- This helps skip data parts that definitely don't match
SELECT count() FROM events WHERE event_type = 'purchase';
```

## Summary

MongoDB developers moving to ClickHouse must shift from document-centric thinking to flat relational schemas. The aggregation pipeline maps naturally to SQL GROUP BY queries, and ClickHouse's columnar storage delivers 10-100x faster analytical query performance compared to MongoDB on large datasets.
