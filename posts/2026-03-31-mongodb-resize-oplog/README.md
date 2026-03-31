# How to Resize the Oplog in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Replica Set, Oplog, Administration, Database

Description: Learn how to resize the MongoDB oplog on a running replica set using replSetResizeOplog, and understand when and why you need a larger oplog.

---

The oplog (operations log) is a capped collection that records all write operations on a MongoDB primary. Secondaries replay the oplog to stay in sync. If the oplog is too small, secondaries can fall behind and become stale, requiring a full resync. This guide explains how to resize the oplog without downtime.

## Why Resize the Oplog?

- **Heavy write workloads** fill the oplog quickly, shrinking the replication window
- **Slow secondaries** (analytics, cross-region) need a larger oplog buffer to catch up
- **Maintenance windows** require enough oplog time to apply updates without resyncing
- A good rule of thumb: oplog should hold at least 24-72 hours of writes

## Check Current Oplog Size

```javascript
rs.printReplicationInfo()
```

Output:

```text
configured oplog size:   2048 MB
log length start to end: 3600 secs (1.0 hrs)
oplog first event time:  ...
oplog last event time:   ...
now:                     ...
```

If "log length" is less than a few hours under normal load, consider increasing the size.

## Check Oplog Details

```javascript
use local
db.oplog.rs.stats().maxSize
```

## Resize on a Running Primary (MongoDB 3.6+)

Use the `replSetResizeOplog` command to resize without restarting:

```javascript
db.adminCommand({ replSetResizeOplog: 1, size: 10240 })
```

`size` is in megabytes. This example sets the oplog to 10 GB.

The change takes effect immediately on the running instance but is not persisted to the config. To make it permanent, set it in `mongod.conf`:

```yaml
replication:
  replSetName: "rs0"
  oplogSizeMB: 10240
```

## Resize on All Replica Set Members

Run the command on each member, starting with secondaries then the primary:

```bash
# On each secondary
mongosh --host mongo2:27017 --eval 'db.adminCommand({ replSetResizeOplog: 1, size: 10240 })'
mongosh --host mongo3:27017 --eval 'db.adminCommand({ replSetResizeOplog: 1, size: 10240 })'

# Then on the primary
mongosh --host mongo1:27017 --eval 'db.adminCommand({ replSetResizeOplog: 1, size: 10240 })'
```

## Reduce Oplog Size

You can also shrink the oplog. The `minRetentionHours` parameter prevents the oplog from being truncated below a minimum age:

```javascript
db.adminCommand({
  replSetResizeOplog: 1,
  size: 2048,
  minRetentionHours: 1
})
```

This is useful for storage-constrained environments while ensuring at least 1 hour of oplog is retained.

## Monitor Replication Lag

After resizing, confirm secondaries are catching up:

```javascript
rs.printSecondaryReplicationInfo()
```

Output shows each secondary's lag in seconds. A value over a few minutes warrants investigation.

## Verify the New Size

```javascript
use local
db.oplog.rs.stats().maxSize / (1024 * 1024) + " MB"
```

## Summary

The oplog window is a critical operational parameter for MongoDB replica sets. Use `replSetResizeOplog` for a live resize without a restart, and persist the change in `mongod.conf` so it survives restarts. Aim for an oplog large enough to cover at least 24 hours of writes, and monitor `rs.printSecondaryReplicationInfo()` regularly to catch lag before secondaries fall too far behind.

