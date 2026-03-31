# How to Troubleshoot MongoDB Sharding Problems

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Sharding, Troubleshooting, Balancer, Chunk

Description: Diagnose and resolve MongoDB sharding problems including balancer issues, uneven chunk distribution, stale routing, and config server failures.

---

## Overview

MongoDB sharding problems commonly manifest as uneven data distribution, balancer not running, queries hitting all shards (scatter-gather), stale routing metadata, or config server availability issues. This guide covers systematic diagnosis and resolution for each category.

## Step 1: Check the Overall Sharding Status

```javascript
// Connect to mongos and run the sharding overview
sh.status()

// Key things to look for:
// - balancer.currently running
// - Number of chunks per shard (should be roughly equal)
// - Shards listed as active
```

## Step 2: Diagnose Balancer Issues

The balancer moves chunks between shards to equalize distribution.

```javascript
// Check if balancer is enabled
sh.getBalancerState()

// Check if balancer is currently running
sh.isBalancerRunning()

// Check for balancer errors in recent rounds
use config
db.actionlog.find({ what: 'balancer.round' }).sort({ time: -1 }).limit(5)

// Check for failed migrations
db.changelog.find({ what: /moveChunk/ , 'details.ok': 0 }).sort({ time: -1 }).limit(10)
```

Enable the balancer if it was accidentally disabled.

```javascript
sh.startBalancer()
```

## Step 3: Diagnose Uneven Chunk Distribution

```javascript
// Count chunks per shard for a specific collection
use config
db.chunks.aggregate([
  { $match: { ns: 'mydb.orders' } },
  { $group: { _id: '$shard', count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])
```

If one shard has significantly more chunks than others and the balancer is enabled, the shard key may be causing a hotspot. Check for jumbo chunks.

```javascript
// Find jumbo chunks (too large to split and move)
db.chunks.find({ ns: 'mydb.orders', jumbo: true }).count()

// Clear jumbo flag manually (only after ensuring the chunk can be split)
db.chunks.updateOne(
  { ns: 'mydb.orders', min: { userId: 'someValue' } },
  { $unset: { jumbo: 1 } }
)

// Then manually split the chunk
sh.splitAt('mydb.orders', { userId: 'someMidpointValue' })
```

## Step 4: Identify Scatter-Gather Queries

Queries that do not include the shard key hit every shard and are expensive.

```javascript
// Use explain to see which shards are targeted
db.orders.find({ status: 'pending' }).explain('executionStats')

// Look for "winningPlan.stage" = "SHARD_MERGE" and count of shardsQueried
// If shardsQueried equals total shards, the query is a scatter-gather
```

Fix scatter-gather by including the shard key in the query filter.

```javascript
// Scatter-gather (bad) - no shard key
db.orders.find({ status: 'pending' })

// Targeted (good) - includes shard key
db.orders.find({ customerId: 'cust123', status: 'pending' })
```

## Step 5: Flush Stale Routing Cache

Application servers and mongos instances cache routing tables. Stale routing causes `StaleConfig` errors.

```javascript
// Flush the routing cache on the mongos you are connected to
db.adminCommand({ flushRouterConfig: 1 })

// Or for a specific namespace
db.adminCommand({ flushRouterConfig: 'mydb.orders' })
```

## Step 6: Check Config Server Health

```bash
# Config servers must be a replica set (CSRS)
# Connect to the config server and check replica set status
mongosh --host configsvr1:27019 --eval 'rs.status()'

# Look for members with stateStr other than PRIMARY or SECONDARY
# A config server in RECOVERING state will cause write failures on mongos
```

## Step 7: Check for Orphaned Documents

After failed chunk migrations, orphaned documents can exist on the wrong shard.

```javascript
// Run cleanupOrphaned on the shard (run on each shard, not mongos)
db.adminCommand({
  cleanupOrphaned: 'mydb.orders',
  startingFromKey: { userId: MinKey }
})
```

## Summary

MongoDB sharding troubleshooting starts with `sh.status()` for an overview, then progresses to balancer health, chunk distribution, and query targeting. Fix hotspots by splitting jumbo chunks, resolve scatter-gather queries by including the shard key in filters, flush stale routing with `flushRouterConfig`, and run `cleanupOrphaned` to remove stale documents after failed migrations.
