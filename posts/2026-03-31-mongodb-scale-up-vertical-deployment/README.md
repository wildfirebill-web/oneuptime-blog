# How to Scale Up (Vertical) a MongoDB Deployment

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Scaling, Performance, Operations, Replica Set

Description: Learn how to vertically scale a MongoDB deployment by increasing CPU, memory, or storage on existing nodes using a rolling approach to avoid downtime.

---

Vertical scaling - adding more CPU, RAM, or faster storage to existing servers - is often the fastest way to improve MongoDB performance when query patterns are already optimized. The rolling approach lets you upgrade each node without taking the cluster offline.

## When to Scale Vertically

Vertical scaling is appropriate when you observe:
- High CPU utilization during peak query periods
- Frequent page faults indicating insufficient RAM for the working set
- Slow disk I/O causing write or read latency

Check current resource utilization:

```javascript
db.runCommand({ serverStatus: 1 }).mem
db.runCommand({ serverStatus: 1 }).wiredTiger.cache
```

Compare `bytes currently in the cache` to `maximum bytes configured` in WiredTiger stats to detect cache pressure.

## Step 1 - Plan the Upgrade Path

For replica sets, you upgrade secondaries first, then step down the primary. Confirm replica set health before starting:

```javascript
rs.status()
```

All members should be in `PRIMARY` or `SECONDARY` state with minimal replication lag.

## Step 2 - Upgrade a Secondary Node

On cloud infrastructure (AWS example), stop the instance and resize:

```bash
# Stop the instance via AWS CLI
aws ec2 stop-instances --instance-ids i-0abc1234

# Modify instance type
aws ec2 modify-instance-attribute --instance-id i-0abc1234 --instance-type "{\"Value\": \"r6i.2xlarge\"}"

# Restart
aws ec2 start-instances --instance-ids i-0abc1234
```

On bare metal, this means physically upgrading RAM or migrating to a larger server, then rejoining the replica set.

After the node restarts, verify it catches up:

```javascript
rs.printSecondaryReplicationInfo()
```

## Step 3 - Tune WiredTiger Cache After Upgrade

After adding RAM, update the WiredTiger cache size in `/etc/mongod.conf`:

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 14
```

A common guideline is to set this to roughly 50% of available RAM, leaving the rest for the OS file system cache and other processes.

Restart MongoDB to apply:

```bash
sudo systemctl restart mongod
```

## Step 4 - Repeat for All Secondaries, Then Primary

After all secondaries are upgraded, step down the primary:

```javascript
rs.stepDown(60)
```

Then upgrade the former primary node following the same process.

## Verify Performance Improvement

After the upgrade, compare key metrics:

```javascript
db.runCommand({ serverStatus: 1 }).opcounters
db.runCommand({ serverStatus: 1 }).globalLock.currentQueue
```

Also review the slow query log for improvements in query execution time.

## Summary

Vertical scaling in MongoDB requires a rolling node-by-node approach to maintain availability. Upgrade secondaries first, adjust WiredTiger cache settings to match the new hardware, wait for replication to catch up, then step down and upgrade the primary. Monitor cache utilization and query latency to confirm the upgrade delivered the expected gains.
