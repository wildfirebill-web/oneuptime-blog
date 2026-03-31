# How to Clean Up Orphaned Documents in Sharded MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Sharding, Administration, Cleanup, Maintenance

Description: Learn how orphaned documents occur in sharded MongoDB clusters and how to clean them up using cleanupOrphaned, cleanupReshardCollection, and rangeDeleter settings.

---

Orphaned documents are documents that exist on a shard but do not belong to that shard according to the cluster's routing metadata. They are a normal byproduct of chunk migrations and MongoDB provides built-in tools to remove them safely.

## Why Orphaned Documents Occur

During a chunk migration, MongoDB moves a chunk from one shard to another. If the migration fails or is interrupted after documents are copied but before the old copies are deleted, orphaned documents remain on the source shard. They do not affect query correctness for routed queries but consume unnecessary disk space and can cause confusing counts when querying individual shards directly.

## Detecting Orphaned Documents

Check if any orphaned ranges exist using `sh.status()`:

```javascript
sh.status()
```

Look for entries with `"jumbo"` flags or ongoing migrations that may have left debris. A more targeted check:

```javascript
use config
db.migrationCoordinators.find({ state: { $ne: "done" } })
```

Also check for pending range deletions:

```javascript
use config
db.rangeDeletions.find()
```

## Automatic Cleanup via rangeDeleter

MongoDB automatically schedules orphan cleanup after chunk migrations via the range deleter. Verify it is enabled and tune its rate:

```javascript
// Check current rangeDeleter settings in mongod.conf
// (no runtime view - check configuration file)
```

In `mongod.conf`:

```yaml
sharding:
  clusterRole: shardsvr
```

The range deleter runs in the background. Monitor its progress:

```javascript
db.adminCommand({ currentOp: true, desc: /range/ })
```

## Manual Cleanup with cleanupOrphaned

For MongoDB 4.x and earlier, use the `cleanupOrphaned` admin command on each shard's primary:

```javascript
// Run on the PRIMARY of each shard, not on mongos
use admin
db.runCommand({
  cleanupOrphaned: "myDatabase.myCollection",
  startingFromKey: { shardKeyField: MinKey },
  secondaryThrottle: false,
  waitForDelete: true
})
```

Repeat with `startingFromKey` set to the `stoppedAtKey` value from the previous response until the cleanup is complete:

```javascript
let result = { stoppedAtKey: { shardKeyField: MinKey } };

while (result.stoppedAtKey) {
  result = db.runCommand({
    cleanupOrphaned: "myDatabase.myCollection",
    startingFromKey: result.stoppedAtKey,
    waitForDelete: true
  });
  print(`Status: ${result.ok}, next: ${tojson(result.stoppedAtKey)}`);
}
print("Cleanup complete");
```

## MongoDB 6.0+ - reshard and Auto-Cleanup

In MongoDB 6.0+, after resharding a collection, cleanup runs automatically. You can also trigger cleanup via:

```javascript
use admin
db.adminCommand({ cleanupStructuredEncryptionData: 1 })
```

For general orphan cleanup in 6.0+, the automated range deleter handles everything after migrations. Manual `cleanupOrphaned` is deprecated but still supported.

## Preventing Orphans

- Ensure the balancer runs during low-traffic windows.
- Monitor migration failures with `sh.isBalancerRunning()` and `sh.getBalancerState()`.
- Avoid stopping `mongod` processes during active chunk migrations.
- Use the `_waitForDelete` option during manual chunk operations.

```javascript
// Check balancer state
sh.isBalancerRunning()
sh.getBalancerState()

// Review recent migration failures
use config
db.changelog.find({ what: "moveChunk.error" }).sort({ time: -1 }).limit(10)
```

## Summary

Orphaned documents result from interrupted chunk migrations. MongoDB's range deleter removes them automatically after successful migrations. For manual cleanup on MongoDB 4.x, use `cleanupOrphaned` iteratively on each shard's primary. On MongoDB 6.0+, the automated cleanup handles post-migration orphans without manual intervention.
