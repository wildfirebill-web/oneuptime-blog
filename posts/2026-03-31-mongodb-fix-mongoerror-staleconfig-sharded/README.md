# How to Fix MongoError: StaleConfig Error in Sharded MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Sharding, Stale Config, Chunk Migration, Error

Description: Learn why StaleConfig errors occur in sharded MongoDB clusters and how to fix them by refreshing routing tables, handling chunk migrations, and tuning balancer settings.

---

## Understanding the Error

`MongoServerError: StaleConfig` (error code 13388) occurs in a sharded cluster when a `mongos` router or shard has stale routing metadata - its cached chunk map does not match the config server's current state. This typically happens after chunk migrations, shard additions, or resharding operations.

```text
MongoServerError: staleConfig: sent epoch does not match current epoch
    code: 13388, codeName: 'StaleConfig'
```

The error is usually transient - the driver automatically retries the operation after refreshing the routing table. If it persists, investigation is needed.

## Step 1: Check If the Error Is Transient

For most applications, `StaleConfig` is handled automatically by the MongoDB driver. Enable retryable reads and writes:

```javascript
const client = new MongoClient(uri, {
  retryWrites: true,
  retryReads: true
});
```

If the error occurs once and then resolves on retry, no further action is needed - it was a normal routing table refresh.

## Step 2: Force Routing Table Refresh

If the error persists, flush the `mongos` routing cache:

```javascript
// Connect to mongos
use admin
db.adminCommand({ flushRouterConfig: 1 })
```

Or flush a specific collection's routing:

```javascript
db.adminCommand({ flushRouterConfig: "mydb.orders" })
```

## Step 3: Check Balancer Activity

The balancer moves chunks between shards. Heavy balancing activity causes frequent `StaleConfig` as chunk boundaries change:

```javascript
// Check balancer status
sh.getBalancerState()
sh.isBalancerRunning()

// Check recent migrations
db.getSiblingDB("config").changelog.find(
  { what: "moveChunk.from" },
  { time: 1, details: 1 }
).sort({ time: -1 }).limit(10)
```

If balancing is too aggressive, configure a balancing window during off-peak hours:

```javascript
use config
db.settings.updateOne(
  { _id: "balancer" },
  {
    $set: {
      activeWindow: {
        start: "02:00",
        stop: "06:00"
      }
    }
  },
  { upsert: true }
)
```

## Step 4: Verify Config Server Reachability

`StaleConfig` can also indicate that `mongos` cannot reach the config server to get the latest routing table:

```bash
# Test connectivity from mongos to config server
nc -zv configserver1 27019
nc -zv configserver2 27019
nc -zv configserver3 27019
```

## Step 5: Check for Version Mismatches

If you recently upgraded some shards but not others, or upgraded `mongos` without upgrading all components, routing metadata versions may conflict:

```javascript
// Check versions of all cluster components
db.adminCommand({ serverStatus: 1 }).version
```

Upgrade all cluster components to the same MongoDB version.

## Step 6: Shard Key Refinement (MongoDB 4.4+)

If you recently refined the shard key using `refineCollectionShardKey`, ensure all `mongos` instances have picked up the new routing:

```javascript
db.adminCommand({ flushRouterConfig: "mydb.orders" })
```

## Monitoring StaleConfig Frequency

```javascript
// Check for stale config errors in server metrics
db.serverStatus().shardingStatistics
```

If `StaleConfig` errors are occurring at a very high rate (not just occasional transient hits), investigate chunk migration patterns and config server latency.

## Summary

`StaleConfig` is a normal part of sharded cluster operation - routing tables periodically go stale after chunk migrations. Ensure retryable reads/writes are enabled so the driver handles it automatically. For persistent issues, flush the routing cache with `flushRouterConfig`, schedule balancer windows during off-peak hours, and verify config server connectivity and version consistency.
