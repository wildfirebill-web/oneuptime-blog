# How to Use CLUSTER SAVECONFIG in Redis to Persist Cluster Config

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Redis Cluster, CLUSTER SAVECONFIG, Configuration, Persistence

Description: Learn how to use CLUSTER SAVECONFIG in Redis to force the node to immediately write its cluster configuration to disk.

---

Redis Cluster nodes maintain a configuration file (typically `nodes.conf`) that records the current cluster state - node IDs, slot assignments, and replication relationships. This file is normally updated automatically, but `CLUSTER SAVECONFIG` lets you force an immediate write to disk.

## Syntax

```text
CLUSTER SAVECONFIG
```

This command takes no arguments and returns `OK` on success.

## When Redis Writes the Config File

Redis Cluster writes the `nodes.conf` file automatically on:
- Cluster state changes (new node joins, failover occurs)
- Server shutdown

However, in some edge cases - particularly after manual cluster changes - the in-memory state may differ from the on-disk config. `CLUSTER SAVECONFIG` forces synchronization.

## Basic Usage

```bash
redis-cli -p 7001 CLUSTER SAVECONFIG
# OK
```

## Common Scenarios

### After Manual Slot Reassignment

When you manually reassign slots using `CLUSTER SETSLOT` or `CLUSTER ADDSLOTS`, save the config immediately:

```bash
redis-cli -p 7001 CLUSTER SETSLOT 500 NODE <node-id>
redis-cli -p 7001 CLUSTER SAVECONFIG
```

### After Adding a New Node

```bash
redis-cli -p 7001 CLUSTER MEET 192.168.1.10 7004
redis-cli -p 7001 CLUSTER SAVECONFIG
```

### Ensuring Config Consistency Across All Nodes

Run `CLUSTER SAVECONFIG` on every node in the cluster as part of a maintenance procedure:

```bash
for port in 7001 7002 7003 7004 7005 7006; do
  redis-cli -p $port CLUSTER SAVECONFIG
  echo "Saved config on port $port"
done
```

## The nodes.conf File

By default, Redis writes cluster config to `nodes.conf` in the working directory. This path is set in `redis.conf`:

```text
cluster-config-file nodes.conf
```

You can inspect this file to verify the cluster state:

```bash
cat /var/lib/redis/nodes.conf
```

The file contains entries like:

```text
<node-id> 127.0.0.1:7001@17001 master - 0 1234567890 1 connected 0-5460
```

## Automating Config Saves

In environments where you perform frequent manual cluster changes, add `CLUSTER SAVECONFIG` to your automation scripts to prevent config drift:

```bash
#!/bin/bash
# After any cluster modification
perform_cluster_change() {
  # ... cluster change commands ...
  redis-cli -p $NODE_PORT CLUSTER SAVECONFIG
}
```

## Summary

`CLUSTER SAVECONFIG` forces Redis Cluster to immediately write its current state to the `nodes.conf` file. While Redis handles most config persistence automatically, running this command after manual cluster modifications ensures the on-disk state matches the in-memory state, protecting you from inconsistencies after unexpected restarts.
