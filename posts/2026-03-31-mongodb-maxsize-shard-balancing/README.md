# How to Configure maxSize for Shard Balancing in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Sharding, Balancer, Capacity Management, Database

Description: Learn how to use the maxSize setting on MongoDB shards to cap storage usage and prevent the balancer from adding chunks to nearly-full shards.

---

In a MongoDB sharded cluster, the balancer moves chunks between shards to keep data evenly distributed. The `maxSize` setting lets you cap the maximum storage size for a shard, preventing the balancer from migrating chunks to it once it reaches that threshold. This is useful when shards have different storage capacities.

## What maxSize Does

`maxSize` is expressed in megabytes. When a shard's storage reaches `maxSize`, the balancer will not migrate additional chunks to it. The shard continues to serve reads and writes for existing data, but it stops receiving new chunks from the balancer.

Note: `maxSize` is a soft limit on balancer behavior, not a hard cap on writes. Direct inserts to a shard that exceed `maxSize` will still succeed.

## Set maxSize on a Shard

Connect to `mongos` and update the shard configuration in the `config` database:

```javascript
use config
db.shards.updateOne(
  { _id: "shard2" },
  { $set: { maxSize: 10240 } }  // 10240 MB = 10 GB
)
```

To view current shard configuration including `maxSize`:

```javascript
db.shards.find().pretty()
```

## Check Current Shard Storage Usage

View how much storage each shard is using:

```javascript
// From mongos
db.adminCommand({ listShards: 1 })
```

Or check via `sh.status()` which includes shard data sizes.

## Add a Shard with maxSize via the addShard Command

When adding a new shard to the cluster, set `maxSize` at the time of addition:

```javascript
db.adminCommand({
  addShard: "shard3RS/shard3.example.com:27018",
  maxSize: 20480   // 20 GB
})
```

## Remove the maxSize Limit

To remove the `maxSize` restriction and allow the balancer to use a shard freely:

```javascript
use config
db.shards.updateOne(
  { _id: "shard2" },
  { $unset: { maxSize: "" } }
)
```

## Verify Balancer Respects maxSize

After setting `maxSize`, run the balancer and watch the chunk counts:

```javascript
sh.startBalancer()

// Wait a few minutes, then check chunk distribution
use config
db.chunks.aggregate([
  { $group: { _id: "$shard", count: { $sum: 1 } } }
])
```

The shard with `maxSize` set should stop growing in chunk count once it approaches the limit.

## Practical Use Case: Mixed Capacity Shards

If you have two shards - one with 20 GB of disk and one with 50 GB - configure `maxSize` to prevent the balancer from overfilling the smaller shard:

```javascript
// Limit the smaller shard to 18 GB (leaving headroom)
use config
db.shards.updateOne({ _id: "shard-small" }, { $set: { maxSize: 18432 } })
```

The 50 GB shard has no limit and will absorb more chunks as the cluster grows.

## Summary

The `maxSize` setting in MongoDB sharding gives you control over how the balancer distributes chunks across shards with different storage capacities. Set it in the `config.shards` collection to prevent a shard from being overloaded, and remove or adjust it as hardware is upgraded. Always leave headroom - set `maxSize` below the actual disk capacity to avoid running out of space.
