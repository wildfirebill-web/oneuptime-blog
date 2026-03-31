# MongoDB vs ClickHouse: OLTP vs OLAP Comparison

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, ClickHouse, OLAP, OLTP, Comparison

Description: Compare MongoDB and ClickHouse across their OLTP vs OLAP strengths, query patterns, storage engines, and when to use each or combine them together.

---

## Overview

MongoDB is an OLTP (Online Transaction Processing) database built for transactional reads and writes on individual documents. ClickHouse is an OLAP (Online Analytical Processing) columnar database built for aggregating billions of rows at blazing speed. They solve different problems and are often used together.

## Storage Architecture

MongoDB uses a row-oriented storage engine (WiredTiger). Each document is stored as a complete unit, making single-document reads and writes fast:

```javascript
// MongoDB: efficient for point lookups and document-level writes
db.users.findOne({ _id: userId })
db.orders.updateOne({ _id: orderId }, { $set: { status: "shipped" } })
```

ClickHouse uses a columnar storage engine. It reads only the columns needed for a query, making aggregations over millions of rows extremely fast:

```sql
-- ClickHouse: efficient for aggregate analytics
SELECT
    toDate(created_at) AS day,
    product_category,
    sum(revenue) AS total_revenue,
    count() AS order_count
FROM orders
WHERE created_at >= today() - 30
GROUP BY day, product_category
ORDER BY day DESC, total_revenue DESC
```

## Write Patterns

MongoDB handles individual document inserts and updates efficiently with low latency. ClickHouse is optimized for bulk inserts and is not well-suited to high-frequency single-row updates.

```python
# ClickHouse: insert in batches for best performance
import clickhouse_connect

client = clickhouse_connect.get_client(host="localhost")
client.insert("events", [
    ["user-1", "page_view", "2024-01-15 10:00:00"],
    ["user-2", "purchase", "2024-01-15 10:01:00"],
    # ... thousands more rows
], column_names=["user_id", "event_type", "ts"])
```

## Using Both Together

A common pattern is to use MongoDB for transactional data and stream it to ClickHouse for analytics:

```bash
# Use Kafka Connect to stream MongoDB change events to ClickHouse
# 1. MongoDB -> Kafka via Debezium connector
# 2. Kafka -> ClickHouse via ClickHouse Kafka engine or Kafka Connect ClickHouse sink
```

## Query Comparison

```javascript
// MongoDB: aggregation on 10 million rows is slow without proper indexes
db.events.aggregate([
  { $match: { ts: { $gte: ISODate("2024-01-01") } } },
  { $group: { _id: "$eventType", count: { $sum: 1 } } }
])
```

```sql
-- ClickHouse: same query on 10 billion rows in milliseconds
SELECT event_type, count() AS cnt
FROM events
WHERE ts >= '2024-01-01'
GROUP BY event_type
ORDER BY cnt DESC
```

## When to Use MongoDB

- Transactional workloads: creating, updating, and retrieving individual records
- Applications with flexible, evolving schemas
- Document storage with nested structures and variable fields

## When to Use ClickHouse

- Analytical dashboards aggregating millions to billions of rows
- Time-series analytics, log analysis, and clickstream data
- Reporting queries that scan entire datasets with GROUP BY and aggregations

## Summary

MongoDB and ClickHouse are complementary rather than competing technologies. Use MongoDB for your application's transactional data layer and ClickHouse as your analytics layer. Stream data from MongoDB to ClickHouse using Kafka or change data capture to get the best of both worlds - fast OLTP writes and millisecond OLAP query performance.
