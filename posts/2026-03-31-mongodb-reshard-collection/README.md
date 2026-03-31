# How to Reshard a Collection in MongoDB 5.0+

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Sharding, Resharding, Shard Key, Database

Description: Learn how to use reshardCollection in MongoDB 5.0+ to change a collection shard key online without downtime, and what to expect during the resharding process.

---

MongoDB 5.0 introduced online resharding, which lets you change the shard key for a collection while the cluster continues serving reads and writes. This eliminates the need to dump and reload data when the original shard key choice proves suboptimal.

## When to Reshard

Consider resharding when:
- The current shard key creates hotspot writes (e.g., monotonically increasing `_id`)
- The cluster is severely imbalanced and refinement cannot fix it
- Query patterns have changed and the shard key no longer aligns with them

## Prerequisites

- MongoDB 5.0 or later
- Sufficient disk space on each shard (roughly equal to the collection size)
- A valid shard key candidate that provides good cardinality and distribution

## Run reshardCollection

Connect to `mongos` and run the command:

```javascript
db.adminCommand({
  reshardCollection: "myapp.orders",
  key: { region: 1, customerId: 1 }
})
```

MongoDB starts the resharding operation in the background. The command returns once resharding begins, not when it completes.

## Monitor Resharding Progress

Track progress using `currentOp`:

```javascript
db.adminCommand({
  currentOp: true,
  type: "op",
  "command.reshardCollection": "myapp.orders"
})
```

Or use the `$currentOp` aggregation:

```javascript
db.adminCommand({
  aggregate: 1,
  pipeline: [
    { $currentOp: { allUsers: true, idleConnections: true } },
    { $match: { type: "op", "command.reshardCollection": { $exists: true } } }
  ],
  cursor: {}
})
```

Look for `remainingOperationTimeEstimatedSecs` in the output.

## What Happens During Resharding

1. MongoDB creates temporary "recipient" shards based on the new shard key.
2. Data is cloned from the existing shards to the recipients.
3. Change streams capture all writes during the clone phase and apply them to the recipients.
4. When the clone and catch-up are complete, MongoDB cuts over to the new shard key in a brief window (typically under a second).
5. The old shard key chunks are deleted.

The collection remains fully readable and writable throughout the process.

## Abort Resharding

If you need to cancel a resharding operation in progress:

```javascript
db.adminCommand({ abortReshardCollection: "myapp.orders" })
```

The collection reverts to its original shard key with no data loss.

## Commit Resharding (if using commitQuorum)

By default, MongoDB automatically commits resharding. You can require a manual commit with `commitQuorum: "majority"` and then commit explicitly:

```javascript
db.adminCommand({ commitReshardCollection: "myapp.orders" })
```

## Verify the New Shard Key

After completion, verify the new shard key is active:

```javascript
use config
db.collections.findOne({ _id: "myapp.orders" })
// "key" should show { region: 1, customerId: 1 }

sh.status()
```

## Performance Considerations

Resharding is I/O and network intensive. Schedule it during low-traffic periods or throttle it:

```javascript
db.adminCommand({
  reshardCollection: "myapp.orders",
  key: { region: 1, customerId: 1 },
  _presetReshardedChunks: []  // Optional: specify chunk distribution
})
```

Monitor disk usage on all shards during resharding to avoid running out of space.

## Summary

`reshardCollection` in MongoDB 5.0+ enables online shard key changes without downtime. It copies data to new shards, applies in-flight changes via change streams, and atomically cuts over to the new shard key. Monitor progress with `currentOp`, ensure sufficient disk space, and use `abortReshardCollection` to safely cancel if needed.
