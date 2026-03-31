# How to Create a Hashed Index for Hash-Based Sharding in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Indexes, Hashed Index, Sharding, Database

Description: Learn how to create hashed indexes in MongoDB for hash-based sharding, enabling even data distribution across shards for write-heavy workloads.

---

## Overview

A hashed index in MongoDB indexes the hash of a field's value rather than the value itself. Hashed indexes are primarily used as shard keys for hash-based sharding, which distributes documents evenly across shards by hashing the shard key value. This prevents hotspots in range-based sharding patterns where sequential values would route to the same shard.

## Creating a Hashed Index

Use `"hashed"` as the index type:

```javascript
db.orders.createIndex({ orderId: "hashed" })
```

## Using a Hashed Index as a Shard Key

The most common use case is enabling hash-based sharding when setting up a sharded collection:

```javascript
// First, enable sharding on the database
sh.enableSharding("mydb")

// Create the hashed index on the shard key field
db.orders.createIndex({ customerId: "hashed" })

// Shard the collection using hashed sharding
sh.shardCollection("mydb.orders", { customerId: "hashed" })
```

With hashed sharding, documents are distributed based on the hash of `customerId`, ensuring roughly equal distribution even when customers are added sequentially.

## Hash-Based vs Range-Based Sharding

```text
Range-based sharding:
- Shard key values are divided into contiguous ranges
- Good for range queries (find all orders in a date range)
- Risk of hotspots with sequential shard keys (auto-increment IDs, timestamps)

Hash-based sharding:
- Shard key values are hashed to determine shard placement
- Even distribution regardless of value patterns
- Scatter-gather queries (range queries require contacting all shards)
- Best for high write throughput workloads with no range query needs
```

## Querying with a Hashed Index

Hashed indexes only support equality queries, not range or sort operations:

```javascript
// Supported - equality query
db.orders.find({ customerId: "user_123" })

// NOT supported for range queries using hashed index
// db.orders.find({ customerId: { $gt: "user_100" } }) // Will do collection scan
```

## Limitations of Hashed Indexes

Important constraints to know:

```text
- Only support equality queries (no range, sort, or $in benefit)
- Cannot create a unique hashed index
- Only one hashed field per compound index (compound hashed indexes are supported in MongoDB 4.4+)
- Floating point values are rounded to 64-bit integers before hashing (loss of precision for very large floats)
```

## Compound Hashed Indexes (MongoDB 4.4+)

MongoDB 4.4 added support for compound hashed indexes where one field is hashed:

```javascript
// One regular field + one hashed field
db.events.createIndex({ type: 1, userId: "hashed" })
```

This can be used as a compound shard key:

```javascript
sh.shardCollection("mydb.events", { type: 1, userId: "hashed" })
```

This distributes data evenly within each event type, useful when events have different write rates per type.

## Verifying Shard Distribution

After sharding, check how data is distributed across shards:

```javascript
sh.status()
db.orders.getShardDistribution()
```

## When to Use Hashed Indexes

```text
Good use cases:
- Collections with sequential or monotonically increasing shard keys
- High-throughput write workloads needing even shard distribution
- Collections where the application primarily queries by exact shard key value
- User ID or session ID-based sharding

Poor use cases:
- Workloads needing range queries on the shard key
- Time-series data that you want to query by time range across all shards efficiently
```

## Summary

Hashed indexes in MongoDB hash field values to create evenly distributed index entries, making them ideal for hash-based sharding of high-write-throughput collections. They prevent shard hotspots caused by sequential shard keys but sacrifice range query efficiency - hashed sharding requires scatter-gather operations for range queries. Create a hashed index before sharding a collection and choose hash-based sharding when uniform write distribution matters more than range query efficiency.
