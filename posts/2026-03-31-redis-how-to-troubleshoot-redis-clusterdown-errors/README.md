# How to Troubleshoot Redis CLUSTERDOWN Errors

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cluster, CLUSTERDOWN, Troubleshooting, High Availability

Description: Diagnose and fix Redis CLUSTERDOWN errors caused by failed nodes, missing hash slots, quorum loss, or misconfigured cluster state.

---

## What is the CLUSTERDOWN Error

Redis Cluster marks itself as down and returns this error when it cannot guarantee data consistency:

```text
CLUSTERDOWN The cluster is down
```

This happens when one or more hash slot ranges have no available primary (all primaries for a slot range have failed and there are not enough replicas to elect a new primary).

## Step 1 - Check Overall Cluster State

```bash
redis-cli -h <any-node> -p 6379 CLUSTER INFO | grep cluster_state
```

Healthy output:

```text
cluster_state:ok
```

Degraded output:

```text
cluster_state:fail
cluster_slots_fail:1
cluster_slots_pfail:0
```

`cluster_slots_fail` shows how many slots are unavailable.

## Step 2 - Identify Failed Nodes

```bash
redis-cli -h <any-node> -p 6379 CLUSTER NODES
```

Look for nodes marked `fail` or `fail?`:

```text
a1b2... 192.168.1.10:6379@16379 master - 0 1711900000 1 connected 0-5460
c3d4... 192.168.1.11:6379@16379 master,fail - 1711899000 1711899050 2 disconnected 5461-10922
e5f6... 192.168.1.12:6379@16379 master - 0 1711900000 3 connected 10923-16383
```

A node marked `fail` has been voted down by the cluster. A node marked `fail?` is suspected but not yet confirmed as failed.

## Step 3 - Restart Failed Nodes

If the failed node is temporarily unavailable (crash, OOM kill), restart it:

```bash
systemctl restart redis@<instance>
# or
redis-server /etc/redis/redis-<port>.conf
```

Once the node rejoins, the cluster will resync and the state will return to `ok`. Monitor:

```bash
watch -n2 "redis-cli -h <any-node> -p 6379 CLUSTER INFO | grep cluster_state"
```

## Step 4 - Manually Trigger Failover

If the failed primary cannot be recovered and has a replica, trigger a manual failover:

```bash
# On the replica that should become the new primary
redis-cli -h <replica-host> -p <replica-port> CLUSTER FAILOVER
```

For a hard failover when the primary is unreachable:

```bash
redis-cli -h <replica-host> -p <replica-port> CLUSTER FAILOVER FORCE
```

After the failover, check cluster nodes again:

```bash
redis-cli -h <any-node> -p 6379 CLUSTER NODES | grep master
```

## Step 5 - Handle Missing Replicas (cluster-require-full-coverage)

By default, if any slot range loses all its nodes, the entire cluster goes down (`cluster-require-full-coverage yes`). If you prefer the cluster to serve the slots that are available, disable this setting:

```bash
redis-cli -h <node> -p 6379 CONFIG SET cluster-require-full-coverage no
```

This allows the cluster to remain up for available slots while the failed slot range is down. Apply on all nodes.

## Step 6 - Add a Replacement Node

If the failed node cannot be recovered:

```bash
# Start a new Redis instance
redis-server /etc/redis/redis-6380.conf

# Add it to the cluster as a replica
redis-cli --cluster add-node <new-node-ip>:6380 <existing-node-ip>:6379 --cluster-slave --cluster-master-id <failed-master-id>

# Or add as a new primary and rebalance
redis-cli --cluster add-node <new-node-ip>:6380 <existing-node-ip>:6379
redis-cli --cluster rebalance <existing-node-ip>:6379
```

## Step 7 - Fix Cluster After Network Partition

After a network partition, nodes on each side may have diverged. When they reconnect, check for nodes in `handshake` state:

```bash
redis-cli -h <node> CLUSTER NODES | grep handshake
```

If nodes are stuck in `handshake`, reset and re-add them:

```bash
redis-cli -h <stuck-node> CLUSTER RESET SOFT
redis-cli --cluster add-node <stuck-node>:6379 <existing-node>:6379
```

## Step 8 - Verify Cluster Integrity

After recovery:

```bash
redis-cli --cluster check <any-node>:6379
```

This reports slot coverage, node connectivity, and any inconsistencies.

## Summary

Redis CLUSTERDOWN errors occur when hash slot ranges lose all their primary nodes. Start by identifying failed nodes with `CLUSTER NODES`, restart them if possible, or trigger a manual failover to a replica. For production clusters, always maintain at least one replica per shard and monitor cluster state continuously to catch `fail?` states before they become `fail`.
