# How to Use CLUSTER NODES in Redis to View Cluster Topology

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cluster, Topology, Node, Operation

Description: Learn how to use CLUSTER NODES in Redis to inspect the full cluster topology, node roles, slot assignments, and replication relationships.

---

## What Is CLUSTER NODES?

`CLUSTER NODES` is a Redis command that returns a detailed description of all nodes in the cluster, including their IDs, IP addresses, ports, roles, slot ranges, and replication links. It is the primary way to inspect the current topology of a Redis Cluster.

The output is a plain-text table where each line represents one cluster node.

## Basic Usage

```bash
CLUSTER NODES
```

Sample output:

```text
a1b2c3d4e5f6... 192.168.1.10:6379@16379 master - 0 1700000000000 1 connected 0-5460
b2c3d4e5f6a1... 192.168.1.11:6379@16379 master - 0 1700000000001 2 connected 5461-10922
c3d4e5f6a1b2... 192.168.1.12:6379@16379 master - 0 1700000000002 3 connected 10923-16383
d4e5f6a1b2c3... 192.168.1.13:6379@16379 slave a1b2c3d4e5f6... 0 1700000000003 1 connected
```

## Understanding the Output Fields

Each line in `CLUSTER NODES` output contains the following fields in order:

```text
<node-id> <ip:port@bus-port> <flags> <master-id> <ping-sent> <pong-recv> <config-epoch> <link-state> <slot-ranges...>
```

| Field | Description |
|---|---|
| node-id | Unique 40-char hex identifier |
| ip:port@bus | Client port and cluster bus port |
| flags | Comma-separated flags (master, slave, myself, fail) |
| master-id | ID of master (for replicas), `-` for masters |
| ping-sent | Timestamp of last PING sent |
| pong-recv | Timestamp of last PONG received |
| config-epoch | Configuration epoch for failover |
| link-state | `connected` or `disconnected` |
| slots | Hash slot ranges assigned to this node |

## Identifying Yourself in the Cluster

The node running the command includes the `myself` flag:

```bash
CLUSTER NODES | grep myself
```

```text
a1b2c3d4... 127.0.0.1:6379@16379 myself,master - 0 0 1 connected 0-5460
```

## Finding Replica Nodes

```bash
# Show only slave/replica nodes
CLUSTER NODES | grep slave
```

```text
d4e5f6a1... 192.168.1.13:6379@16379 slave a1b2c3d4e5f6... 0 1700000000003 1 connected
```

The fourth field (master-id) identifies which master each replica belongs to.

## Checking for Failed Nodes

```bash
# Look for nodes marked as failed
CLUSTER NODES | grep fail
```

If a node has the `fail` or `fail?` flag, it indicates a node that is unreachable or being monitored for failure.

## Parsing CLUSTER NODES with Python

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

nodes_raw = r.cluster('nodes')

for node_id, node_info in nodes_raw.items():
    print(f"Node: {node_id}")
    print(f"  Address: {node_info.get('host')}:{node_info.get('port')}")
    print(f"  Flags: {node_info.get('flags')}")
    print(f"  Slots: {node_info.get('slots')}")
    print()
```

## Checking Slot Coverage

All 16384 hash slots must be assigned for the cluster to be healthy. You can verify slot coverage:

```bash
# Count total slots assigned
CLUSTER NODES | grep master | grep -v fail | awk '{for(i=9;i<=NF;i++) print $i}' | wc -l
```

## Comparing CLUSTER NODES vs CLUSTER INFO

| Command | Purpose |
|---|---|
| CLUSTER NODES | Per-node detail - topology, slots, roles |
| CLUSTER INFO | Cluster-wide summary - state, slot counts, epoch |

Use `CLUSTER NODES` when you need to understand which node owns which slots or identify replication relationships. Use `CLUSTER INFO` for a quick health check.

## Summary

`CLUSTER NODES` is an essential command for Redis Cluster operations, providing a complete picture of the cluster topology including node IDs, roles, slot assignments, and replication links. It is commonly used for debugging cluster issues, verifying slot distribution, and identifying failed or disconnected nodes. Parse the output carefully to distinguish masters from replicas and to confirm all hash slots are covered.
