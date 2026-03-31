# How to Use Hashed Shard Keys in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Sharding, Shard Key, Database, Scalability

Description: Learn when and how to use hashed shard keys in MongoDB for even data distribution, including setup, limitations, and comparison with ranged sharding.

---

Hashed shard keys distribute documents by hashing the shard key field value, producing a near-uniform distribution across shards. This eliminates hotspots caused by sequential inserts but at the cost of efficient range queries on the shard key.

## When to Use Hashed Sharding

- Insert-heavy workloads with monotonically increasing keys (e.g., timestamps, auto-increment IDs)
- Collections where you always query by exact match, never by range on the shard key
- Situations where even data distribution is more important than shard-targeted range queries

## How Hashed Sharding Works

MongoDB hashes the shard key value using a 64-bit MD5 hash and uses the hash to determine chunk placement. Documents with adjacent shard key values end up on different shards:

```text
customerId "C001" -> hash(C001) -> shard2
customerId "C002" -> hash(C002) -> shard1
customerId "C003" -> hash(C003) -> shard3
```

## Setting Up Hashed Sharding

### Step 1 - Enable Sharding on the Database

```javascript
sh.enableSharding("myapp")
```

### Step 2 - Shard the Collection with a Hashed Key

```javascript
sh.shardCollection("myapp.events", { _id: "hashed" })
```

MongoDB automatically creates a hashed index on `_id` if one does not exist.

To hash a different field:

```javascript
sh.shardCollection("myapp.users", { userId: "hashed" })
```

First create the index manually if the collection has existing data:

```javascript
db.users.createIndex({ userId: "hashed" })
sh.shardCollection("myapp.users", { userId: "hashed" })
```

## Pre-Splitting with Hashed Keys

For empty collections being pre-filled with large datasets, pre-split to distribute immediately:

```javascript
sh.shardCollection("myapp.logs", { _id: "hashed" }, false, { numInitialChunks: 8 })
```

`numInitialChunks` creates N initial chunks distributed across shards, avoiding balancer overhead at import time.

## Verify Distribution

```javascript
db.events.getShardDistribution()
```

Good output shows roughly equal document counts per shard:

```text
Shard shard1 at shard1/mongo-s1:27017
 data : 98.5MiB docs : 520000 chunks : 4
Shard shard2 at shard2/mongo-s2:27017
 data : 99.1MiB docs : 524000 chunks : 4
```

## Limitations of Hashed Sharding

**No range query optimization:**

```javascript
// This is a scatter-gather query (hits all shards)
db.events.find({ _id: { $gt: ObjectId("..."), $lt: ObjectId("...") } })
```

With a hashed `_id`, the range on the original value does not map to a range on the hash, so MongoDB cannot target a specific shard.

**No sort optimization on shard key:**

```javascript
// Cannot use the shard key for sort targeting with hashed keys
db.events.find().sort({ _id: 1 })
```

**Compound hashed keys (one hashed field only):**

```javascript
// Valid: compound key with hashed prefix
sh.shardCollection("myapp.logs", { ts: "hashed", level: 1 })

// NOT valid: hashed field cannot be in the middle or end
// sh.shardCollection("myapp.logs", { level: 1, ts: "hashed" })
```

## Hashed vs Ranged Sharding Comparison

| Criteria | Hashed | Ranged |
|---|---|---|
| Data distribution | Uniform | Can be uneven |
| Insert hotspots | No | Yes (with monotonic keys) |
| Range query efficiency | No (scatter-gather) | Yes (targeted) |
| Best for | Write-heavy, monotonic keys | Range queries, geographic locality |

## Summary

Hashed shard keys provide uniform data distribution and eliminate insert hotspots caused by monotonically increasing fields like timestamps and ObjectIds. Use `numInitialChunks` when pre-populating large collections to avoid initial balancer thrash. The key trade-off is that range queries on the shard key become scatter-gather operations, so hashed sharding is best for workloads that query by exact key match rather than by range.

