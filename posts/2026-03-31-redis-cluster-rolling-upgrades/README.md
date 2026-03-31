# How to Perform Rolling Upgrades in Redis Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cluster, Upgrade, Rolling, Operation

Description: Learn how to upgrade Redis Cluster nodes one at a time using a rolling upgrade strategy to minimize downtime and maintain availability throughout.

---

Rolling upgrades allow you to upgrade each node in a Redis Cluster sequentially while keeping the cluster available. The key is upgrading replicas first, then promoting each replica before upgrading its current primary.

## Before You Start

```bash
# Document current topology
redis-cli -p 7001 CLUSTER NODES > cluster-nodes-before.txt

# Verify cluster is healthy
redis-cli -p 7001 CLUSTER INFO | grep cluster_state
# cluster_state:ok

# Check all replicas are caught up
redis-cli -p 7001 CLUSTER NODES | grep slave
```

Never start an upgrade if the cluster is in a degraded state.

## Rolling Upgrade Sequence

For a 3-primary, 3-replica cluster:

```text
Step 1: Upgrade replica nodes (no slot ownership, safe to restart)
Step 2: Failover primary to its upgraded replica
Step 3: Upgrade the demoted primary (now a replica)
Step 4: Repeat for each shard
```

## Step 1 - Upgrade a Replica

```bash
# Stop the replica
redis-cli -h replica-1 -p 7004 SHUTDOWN NOSAVE
# or gracefully:
redis-cli -h replica-1 -p 7004 SHUTDOWN SAVE

# Install new Redis version
apt-get install redis-server=7.4.0-1

# Start with same config
redis-server /etc/redis/cluster/7004/redis.conf

# Wait for replica to sync
redis-cli -h replica-1 -p 7004 INFO replication | grep master_link_status
# master_link_status:up
```

## Step 2 - Failover Primary to Upgraded Replica

```bash
# Trigger failover - promotes the replica, demotes the old primary
redis-cli -h replica-1 -p 7004 CLUSTER FAILOVER

# Watch for failover completion
redis-cli -p 7001 CLUSTER NODES | grep -E "7001|7004"
```

The `CLUSTER FAILOVER` command (without `FORCE`) waits for the replica to be fully caught up before switching - zero data loss.

```bash
# Verify the upgrade replica is now primary
redis-cli -h replica-1 -p 7004 ROLE
# master
```

## Step 3 - Upgrade the Demoted Node

```bash
# The old primary is now a replica, safe to upgrade
redis-cli -h old-primary-1 -p 7001 SHUTDOWN SAVE

# Install new version
apt-get install redis-server=7.4.0-1

# Restart
redis-server /etc/redis/cluster/7001/redis.conf

# Verify it rejoins as replica
redis-cli -h old-primary-1 -p 7001 INFO replication | grep role
# role:slave
```

## Repeat for All Shards

```bash
# Shard 2: upgrade replica 7005, failover primary 7002, upgrade 7002
redis-cli -h replica-2 -p 7005 SHUTDOWN SAVE
# [upgrade, restart, wait for sync]
redis-cli -h replica-2 -p 7005 CLUSTER FAILOVER
# [verify]
redis-cli -h old-primary-2 -p 7002 SHUTDOWN SAVE
# [upgrade, restart]

# Shard 3: same pattern
```

## Verification After Each Shard

```bash
redis-cli -p 7001 CLUSTER INFO | grep -E "cluster_state|cluster_slots"
redis-cli --cluster check 127.0.0.1:7001
```

## Rollback Plan

If something goes wrong during upgrade:

```bash
# Revert to old version
apt-get install redis-server=7.2.0-1

# Restart with same config
redis-server /etc/redis/cluster/7001/redis.conf

# Trigger failback if needed
redis-cli -h old-node -p 7001 CLUSTER FAILOVER
```

## Version Compatibility

Redis supports rolling upgrades between adjacent major versions. Always check the Redis release notes for compatibility caveats:

```bash
redis-cli -p 7001 INFO server | grep redis_version
```

Keep a mixed-version cluster state as brief as possible.

## Summary

Redis Cluster rolling upgrades follow a replica-first pattern: upgrade each replica, use `CLUSTER FAILOVER` to promote it to primary, then upgrade the demoted node. This maintains availability throughout. Always verify cluster health between each shard upgrade and keep a documented rollback procedure ready.
