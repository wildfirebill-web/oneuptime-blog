# How to Set Up a Delayed Replica Set Member in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Replica Set, Delayed Member, Disaster Recovery, Operation

Description: Configure a delayed MongoDB replica set member that lags behind the primary by a fixed time window to protect against accidental data loss.

---

## Why Use a Delayed Replica Set Member

A delayed member intentionally lags behind the primary by a configured number of seconds. If someone accidentally drops a collection or performs a destructive update, the delayed member still holds the pre-disaster state for the duration of the delay window, giving you a rolling point-in-time recovery option.

A delayed member must have `priority: 0` (cannot become primary) and is typically hidden.

## Configure a Delayed Member

Start a `mongod` instance with the replica set name, then add or reconfigure it as a delayed member. The delay is set in seconds.

```javascript
// Connect to primary
rs.add({
  host: "192.168.1.15:27022",
  priority: 0,
  votes: 0,         // non-voting to keep majority logic clean
  hidden: true,
  secondaryDelaySecs: 3600  // 1 hour delay
});
```

For existing members, use `rs.reconfig()`:

```javascript
const cfg = rs.conf();
const idx = cfg.members.findIndex(m => m.host === "192.168.1.15:27022");
cfg.members[idx].priority = 0;
cfg.members[idx].votes = 0;
cfg.members[idx].hidden = true;
cfg.members[idx].secondaryDelaySecs = 3600;
rs.reconfig(cfg);
```

## Verify the Delay Configuration

```javascript
rs.conf().members.filter(m => m.secondaryDelaySecs > 0)
```

Check the replication optime of the delayed member:

```javascript
rs.printSecondaryReplicationInfo()
```

The delayed member should show a lag approximately equal to the configured delay.

## Using the Delayed Member for Recovery

If you need to recover from a destructive operation, connect directly to the delayed member before the delay window expires:

```bash
# Connect directly to delayed member
mongosh --host 192.168.1.15 --port 27022
```

```javascript
// Read the data that was destroyed on primary
db.importantCollection.find().toArray()
```

Then export and re-import to the primary:

```bash
mongodump \
  --host 192.168.1.15:27022 \
  --db mydb \
  --collection importantCollection \
  --out /tmp/recovery

mongorestore \
  --host 192.168.1.10:27017 \
  --db mydb \
  --collection importantCollection \
  /tmp/recovery/mydb/importantCollection.bson
```

## Choosing the Right Delay Window

The delay window should balance recovery capability against storage costs. Common choices:

- 1 hour - catches most accidental drops discovered quickly
- 4 hours - gives time for off-hours incidents to be noticed
- 24 hours - maximum practical delay before replication buffer overhead grows large

```javascript
// 4 hour delay
secondaryDelaySecs: 14400
```

## Summary

A delayed replica set member provides a rolling point-in-time safety net by intentionally lagging replication. Set it with `hidden: true`, `priority: 0`, `votes: 0`, and the desired `secondaryDelaySecs`. When disaster strikes, connect directly to the delayed member to read or export data before the delay window closes. Combine with regular `mongodump` backups for comprehensive protection.
