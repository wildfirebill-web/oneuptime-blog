# MongoDB vs Redis: When to Use Each Database

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Redis, NoSQL, Cache, Database

Description: Understand the architectural differences between MongoDB and Redis to decide when to use each, or how to combine them effectively in production.

---

## Overview

MongoDB and Redis are both NoSQL databases, but they serve fundamentally different roles. MongoDB is a persistent document database optimized for flexible querying and storage. Redis is an in-memory data structure store optimized for sub-millisecond reads, caching, and pub/sub messaging.

## Architecture and Storage

Redis stores all data in RAM by default, making reads and writes extremely fast - typically under 1 ms. Persistence to disk is optional via RDB snapshots or AOF (Append Only File) logging.

MongoDB stores data on disk using the WiredTiger storage engine. Indexes and working sets are cached in memory, but the database size is not limited by RAM.

```text
Redis:
- All data in memory (optional disk persistence)
- Data limited by available RAM
- Typical latency: < 1ms

MongoDB:
- Data on disk, working set in RAM cache
- Data limited by disk capacity
- Typical latency: 1-10ms (indexed queries)
```

## Data Structures

Redis provides rich in-memory data structures beyond simple key-value pairs.

```bash
# Redis - set a string with TTL
SET session:u123 "eyJhbGci..." EX 3600

# Redis - sorted set for leaderboard
ZADD leaderboard 9500 "user:alice"
ZADD leaderboard 8200 "user:bob"
ZRANGE leaderboard 0 9 WITHSCORES REV

# Redis - pub/sub
PUBLISH notifications "order_shipped:o456"
```

MongoDB stores structured documents and supports complex queries, indexes, and aggregations.

```javascript
// MongoDB - store and query an order document
db.orders.insertOne({
  userId: "u123",
  items: [{ sku: "A1", qty: 2 }],
  total: 49.99,
  status: "pending",
  createdAt: new Date()
});

db.orders.find({ userId: "u123", status: "pending" })
  .sort({ createdAt: -1 });
```

## Persistence and Durability

MongoDB provides strong durability guarantees. With write concern `w: "majority"`, writes are confirmed only after replication to a majority of replica set members.

Redis AOF persistence logs every write but introduces latency overhead. RDB snapshots are periodic - recent writes can be lost on crash without AOF enabled.

```bash
# Redis AOF config (redis.conf)
appendonly yes
appendfsync everysec
```

## Common Usage Pattern: Use Both Together

The most effective production pattern uses Redis as a caching layer in front of MongoDB.

```javascript
// Node.js: Redis cache in front of MongoDB
async function getProduct(productId) {
  const cacheKey = `product:${productId}`;
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const product = await db.collection("products").findOne({ _id: productId });
  await redis.setex(cacheKey, 300, JSON.stringify(product)); // cache 5 min
  return product;
}
```

Redis also excels at rate limiting, session storage, queues, and real-time counters - tasks MongoDB can perform but not as efficiently.

## When to Use Redis Alone

Use Redis alone for: session storage, rate limiting counters, pub/sub messaging, leaderboards, short-lived token storage, and caches where eventual persistence is acceptable.

## When to Use MongoDB Alone

Use MongoDB for: primary persistent storage of application data, complex multi-field queries, aggregation pipelines, document-heavy workloads, and when data size exceeds available RAM.

## Summary

Redis and MongoDB are not competitors but complements. Redis handles speed-critical ephemeral data while MongoDB handles durable, queryable persistent storage. Most production applications benefit from using both - Redis as a cache and queue layer, MongoDB as the source of truth.
