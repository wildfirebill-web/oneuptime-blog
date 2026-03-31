# How to Troubleshoot Replication Issues in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Replica Set, Replication, Troubleshooting, Database

Description: A practical guide to diagnosing and fixing common MongoDB replication problems including lag, RECOVERING state, sync failures, and oplog exhaustion.

---

Replication issues in MongoDB range from minor lag to full resync requirements. This guide covers the key diagnostic commands and resolution steps for the most common problems.

## 1. Check Overall Replica Set Health

```javascript
rs.status()
```

Look for members in states other than `PRIMARY` or `SECONDARY`:

- `RECOVERING` - catching up or performing initial sync
- `STARTUP2` - performing initial sync
- `UNKNOWN` - unreachable, likely a network issue
- `ROLLBACK` - rolling back uncommitted writes after re-joining

## 2. Check Replication Lag

```javascript
rs.printSecondaryReplicationInfo()
```

Example output:

```text
source: mongo2:27017
    syncedTo: Thu Mar 31 2026 12:00:00 GMT
    0 secs (0 hrs) behind the primary
source: mongo3:27017
    syncedTo: Thu Mar 31 2026 11:55:00 GMT
    300 secs (0.08 hrs) behind the primary
```

Lag over a few minutes warrants investigation.

## 3. Check Oplog Window

```javascript
rs.printReplicationInfo()
```

If the "log length" is very short (less than a few hours), the oplog is too small. Secondaries that fall behind more than the oplog window require a full resync.

Resize the oplog:

```javascript
db.adminCommand({ replSetResizeOplog: 1, size: 10240 })
```

## 4. Identify the Sync Source

```javascript
rs.status().members.map(m => ({
  host: m.name,
  state: m.stateStr,
  syncSource: m.syncSourceHost
}))
```

If a secondary is syncing from another secondary (chaining) and that secondary is itself lagging, change the sync source:

```javascript
// Run on the lagging secondary
rs.syncFrom("mongo1:27017")
```

## 5. Check for Network Issues

If a member shows `UNKNOWN`:

```bash
# From the problem member, test connectivity to the primary
mongosh --host mongo1:27017 --eval 'db.adminCommand("ping")'

# Check firewall rules
sudo iptables -L -n | grep 27017
```

## 6. Examine MongoDB Logs

```bash
sudo journalctl -u mongod -n 100 --no-pager | grep -i "replset\|replication\|sync\|error"
```

Key error patterns:

```text
too stale to catch up                    # needs resync
getMore cmd failed ... CursorNotFound   # sync cursor expired, will retry
replSetInitiate error ... no replset    # replSetName not configured
```

## 7. Force a Full Resync

If a secondary is too stale (oplog gap):

```bash
# 1. Stop mongod on the secondary
sudo systemctl stop mongod

# 2. Delete the data directory contents
sudo rm -rf /var/lib/mongodb/*

# 3. Start mongod - it will perform initial sync automatically
sudo systemctl start mongod
```

Monitor the resync progress:

```javascript
rs.status().members.find(m => m.name === "mongo3:27017").infoMessage
```

## 8. Monitor Replication Metrics

Use `db.serverStatus()` for detailed replication metrics:

```javascript
db.adminCommand({ serverStatus: 1 }).repl
db.adminCommand({ serverStatus: 1 }).metrics.repl
```

Key metrics:
- `metrics.repl.buffer.count` - ops buffered for application
- `metrics.repl.apply.ops` - ops applied
- `metrics.repl.network.readersCreated` - sync cursor restarts (high value = instability)

## 9. Check Write Concern Errors

If writes are failing with write concern timeouts:

```javascript
db.adminCommand({ replSetGetStatus: 1 }).members.filter(m => m.state !== 1 && m.state !== 2)
```

Any voting member in a non-healthy state reduces available votes and can cause `w: majority` timeouts.

## Summary

Start troubleshooting MongoDB replication by checking `rs.status()` and `rs.printSecondaryReplicationInfo()`. Lag usually points to oplog size, network issues, or chaining. Members stuck in `RECOVERING` often need a larger oplog or a fresh resync. Monitor `db.serverStatus().metrics.repl` for in-depth metrics, and always keep the oplog large enough to cover your longest planned maintenance window.

