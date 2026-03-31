# How to Handle Unbalanced Shards in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Sharding, Balancer, Administration, Chunk

Description: Learn how to identify and resolve unbalanced shard distributions in MongoDB using the balancer, manual chunk moves, and zone sharding.

---

## Overview

MongoDB's balancer migrates chunks between shards to maintain an even data distribution. When the balancer falls behind, is disabled, or has been throttled, shards become unbalanced - some hold far more chunks than others, leading to uneven disk usage and potentially uneven query or write load.

## Checking Shard Balance

```javascript
// Check chunk distribution per shard
db.adminCommand({ listShards: 1 })

// Check chunk counts per shard for a specific collection
use config
db.chunks.aggregate([
  { $match: { ns: 'shop.orders' } },
  { $group: { _id: '$shard', chunkCount: { $sum: 1 } } },
  { $sort: { chunkCount: -1 } },
])
```

Also use the helper:

```javascript
db.orders.getShardDistribution()
```

A healthy cluster shows roughly equal `chunks` counts across shards. An imbalance of more than 8 chunks triggers automatic balancing by default.

## Checking Balancer Status

```javascript
// Is the balancer enabled?
sh.getBalancerState()  // true or false

// Is the balancer currently running a migration?
sh.isBalancerRunning()

// Get full balancer status
sh.status()
```

## Enabling the Balancer

If the balancer is disabled, re-enable it:

```javascript
sh.startBalancer()

// Wait for balancing to complete
sh.awaitBalancerRound()
```

## Configuring the Balancer Window

If the balancer is restricted to a maintenance window, it may not run often enough:

```javascript
// Remove window restriction to allow balancing anytime
db.getSiblingDB('config').settings.updateOne(
  { _id: 'balancer' },
  { $unset: { activeWindow: 1 } }
)

// OR: extend the window significantly
db.getSiblingDB('config').settings.updateOne(
  { _id: 'balancer' },
  { $set: { activeWindow: { start: '00:00', stop: '23:59' } } }
)
```

## Manually Moving a Chunk

If automatic balancing is too slow, move chunks manually:

```javascript
// Find an over-represented chunk on the hot shard
use config
db.chunks.find({ ns: 'shop.orders', shard: 'shard0' })
  .sort({ 'min.tenantId': 1 }).limit(5)

// Move a specific chunk to a less-loaded shard
db.adminCommand({
  moveChunk: 'shop.orders',
  find: { tenantId: 'acme' },
  to: 'shard2'
})
```

## Setting the Chunk Size

Smaller chunks mean finer-grained balancing but more migration overhead. The default is 128 MB:

```javascript
use config
db.settings.updateOne(
  { _id: 'chunksize' },
  { $set: { value: 64 } },  // 64 MB chunks
  { upsert: true }
)
```

Smaller chunk sizes help the balancer rebalance more quickly when a shard has data that spans many natural splits.

## Splitting Jumbo Chunks

A "jumbo" chunk is one that exceeds the chunk size limit and cannot be migrated by the balancer. Identify and split them:

```javascript
// Find jumbo chunks
use config
db.chunks.find({ ns: 'shop.orders', jumbo: true })

// Split a jumbo chunk manually
db.adminCommand({
  split: 'shop.orders',
  find: { tenantId: 'bigcorp' }
})
```

If a chunk is jumbo because all documents share the same shard key value, splitting is impossible - this indicates a shard key cardinality problem that requires a key refinement.

## Preventing Future Imbalance

- Use hashed shard keys to prevent monotonic write patterns.
- Pre-split collections before loading large amounts of data.
- Monitor chunk counts weekly using `config.chunks`.
- Configure balancer windows to allow sufficient time for migrations.

## Summary

Unbalanced shards in MongoDB are addressed by ensuring the balancer is enabled with a sufficient time window, manually moving chunks if automatic balancing is too slow, and splitting jumbo chunks. For structural imbalance caused by poor shard key selection, use `refineCollectionShardKey` or zone sharding to guide placement. Regular monitoring of `config.chunks` and `getShardDistribution()` helps detect imbalance before it impacts performance.
