# How to Restore a Replica Set from Backup in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Replica Set, Backup, Restore, Database

Description: Learn step-by-step how to restore a MongoDB replica set from a mongodump backup or filesystem snapshot, including single-member and full cluster recovery.

---

Restoring a MongoDB replica set from backup requires careful sequencing to bring data back correctly and re-establish replication. This guide covers restoration using `mongodump`/`mongorestore` and filesystem snapshot approaches.

## Restore Strategy Options

1. **Restore one member, resync others** - fastest for large datasets
2. **Restore all members independently** - necessary for point-in-time recovery
3. **Restore to a standalone, then convert** - simplest for small datasets

## Method 1 - Restore Primary, Resync Secondaries

This is the most efficient approach: restore one member, start it as a standalone to avoid triggering elections, then add secondaries that will auto-resync.

### Step 1 - Stop All Replica Set Members

```bash
sudo systemctl stop mongod
```

### Step 2 - Restore Data on the Primary Node

```bash
# Remove existing data
sudo rm -rf /var/lib/mongodb/*

# Restore from mongodump
mongorestore --host localhost:27017 --drop /backup/dump/
```

If you have a filesystem snapshot (e.g., LVM or EBS snapshot), mount it directly to the data directory instead.

### Step 3 - Start as Standalone (Skip Replica Set)

Temporarily comment out the replication config in `mongod.conf`:

```yaml
# replication:
#   replSetName: "rs0"
```

Start mongod and verify data:

```bash
sudo systemctl start mongod
mongosh --eval 'db.adminCommand("listDatabases")'
```

### Step 4 - Restart with Replica Set Config

Re-enable replication in `mongod.conf`:

```yaml
replication:
  replSetName: "rs0"
```

Restart:

```bash
sudo systemctl restart mongod
```

### Step 5 - Re-initiate the Replica Set

```javascript
rs.initiate({
  _id: "rs0",
  members: [{ _id: 0, host: "mongo1:27017" }]
})
```

### Step 6 - Add Secondaries

```javascript
rs.add("mongo2:27017")
rs.add("mongo3:27017")
```

The secondaries will perform an initial sync from the restored primary.

## Method 2 - mongorestore to All Members

For point-in-time consistency, restore the same backup to all members before starting the set:

```bash
# Run on each member
mongorestore --drop /backup/dump/
```

Then initiate the replica set with all three hosts at once:

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017" },
    { _id: 1, host: "mongo2:27017" },
    { _id: 2, host: "mongo3:27017" }
  ]
})
```

## Restoring with Oplog Replay for Point-in-Time Recovery

If your backup includes an oplog dump, replay it to recover to a specific timestamp:

```bash
mongorestore --oplogReplay --oplogLimit "1743379200:1" /backup/dump/
```

`oplogLimit` format is `<unix_timestamp>:<ordinal>`. This restores all ops up to but not including the given timestamp.

## Verify Restoration

```javascript
// Check replica set status
rs.status()

// Check document counts
db.orders.countDocuments()

// Check replication lag
rs.printSecondaryReplicationInfo()
```

## Common Issues

```text
MongoServerError: not running with --replSet
```

The replica set name in `mongod.conf` does not match the backup's config. Ensure `replSetName` is consistent.

```text
Error: this node has not yet been added to a replica set
```

Run `rs.initiate()` or `rs.add()` to register the member.

## Summary

The most efficient replica set restore is to restore one member from backup, start it as a standalone to verify data, then restart it with replication enabled and let secondaries perform initial sync. For point-in-time recovery, use `--oplogReplay` with `mongorestore`. Always verify document counts and replication lag after completing the restore before returning the cluster to production traffic.

