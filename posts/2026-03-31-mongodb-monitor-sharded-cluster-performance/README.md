# How to Monitor Sharded Cluster Performance in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Sharding, Monitoring, Performance, Database

Description: Learn how to monitor performance across a MongoDB sharded cluster using sh.status(), serverStatus, mongostat, and targeted shard-level diagnostics.

---

Monitoring a sharded cluster is more complex than monitoring a standalone instance because performance issues can originate from mongos routers, config servers, individual shards, or the balancer. This guide covers the key commands and metrics to watch.

## Cluster-Wide Overview

```javascript
sh.status()
```

Provides a snapshot of:
- Active shards and their replica set members
- Chunk counts per shard per collection
- Whether the balancer is enabled

## mongos Performance

The `mongos` router is the entry point for all client requests. Check its stats:

```javascript
// Connect to mongos
db.adminCommand({ serverStatus: 1 }).opcounters
```

Key opcounter metrics:
- `insert`, `query`, `update`, `delete` - operation rates
- `getmore` - cursor continuation rate

Check for targeted vs scatter-gather operations:

```javascript
db.adminCommand({ serverStatus: 1 }).shardingStatistics
```

Key field: `totalRequestsWithoutShardKeyInFindAndModify` - high values indicate frequent scatter-gather.

## Per-Shard Performance

SSH to each shard's primary and run:

```javascript
db.adminCommand({ serverStatus: 1 }).opcounters
db.adminCommand({ serverStatus: 1 }).wiredTiger.cache
db.adminCommand({ serverStatus: 1 }).connections
```

A shard with significantly higher operation counts than others indicates an imbalanced shard key (hotspot).

## Using mongostat Across All Shards

```bash
mongostat --host "mongos1:27017" --discover
```

`--discover` shows all cluster members. Look for uneven `insert`/`query` rates across shards.

## Query Performance on Shards

Enable the profiler on a specific shard to capture slow queries:

```javascript
// Connect directly to shard primary
db.setProfilingLevel(1, { slowms: 100 })
db.system.profile.find().sort({ ts: -1 }).limit(10)
```

On `mongos`, use `explain()` to confirm query routing:

```javascript
db.orders.find({ customerId: "C500" }).explain("executionStats")
```

Look for:
- `SHARD_MERGE` with `nShards: 1` = targeted (good)
- `SHARD_MERGE` with `nShards: N` = scatter-gather (investigate)

## Balancer Monitoring

```javascript
// Is the balancer active?
sh.isBalancerRunning()

// Recent migration events
use config
db.changelog.find({ what: "moveChunk.from" }).sort({ time: -1 }).limit(10)

// Failed migrations
db.changelog.find({ what: "moveChunk.from", "details.errmsg": { $exists: true } })
```

High migration frequency may indicate the chunk size is too small or the shard key causes rapid chunk splitting.

## Config Server Performance

Config servers store cluster metadata. Check their query load:

```javascript
// Connect to config server primary
use config
db.serverStatus().opcounters
```

High load on config servers can slow down all mongos operations.

## Key Metrics to Alert On

| Metric | Location | Threshold |
|---|---|---|
| Replication lag | Each shard secondary | > 30s |
| Queued read/write tickets | Each shard | > 10 |
| Scatter-gather ratio | mongos | > 20% of queries |
| Jumbo chunks | config.chunks | Any |
| Chunk imbalance | sh.status() | > 8 chunks difference |
| Connection pool utilization | mongos + shards | > 80% |

## Checking Connection Pools

```javascript
// Check mongos connections
db.adminCommand({ serverStatus: 1 }).connections

// Check shard-level pool from mongos
db.adminCommand({ connPoolStats: 1 })
```

## Summary

Monitoring a sharded MongoDB cluster requires checking multiple components: the mongos routers for scatter-gather rates, individual shards for hotspots, the balancer for migration activity, and config servers for metadata load. Use `mongostat --discover` for a live view of all members, and `explain()` on mongos to confirm queries are targeted. Alert on jumbo chunks, chunk imbalance, and high scatter-gather ratios as early indicators of sharding configuration problems.

