# How to Use CLUSTER FORGET in Redis to Remove Nodes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Cluster, Node, Operation, Maintenance

Description: Learn how to use CLUSTER FORGET in Redis to remove a node from the cluster's node table, completing the decommissioning of cluster members.

---

## What Is CLUSTER FORGET?

`CLUSTER FORGET` removes a node from the current node's internal cluster node table. Once a node is forgotten by all members of the cluster, it is no longer considered part of the cluster. This command must be run on every node in the cluster to fully remove the target node, because each node maintains its own view of the cluster topology.

## Basic Syntax

```text
CLUSTER FORGET node-id
```

Parameter:
- `node-id` - the 40-character unique identifier of the node to remove

## Why CLUSTER FORGET Is Required

The Redis Cluster gossip protocol propagates node information. If you simply shut down a node, other cluster members still remember it and mark it as failed. `CLUSTER FORGET` explicitly removes the entry so it is not gossiped back into the cluster view.

The forget propagation has a 60-second window - if you don't run it on all nodes within 60 seconds, the node may reappear in cluster views (gossip from nodes that still know about it).

## Step-by-Step Node Removal

```bash
# Step 1: Get the ID of the node to remove
redis-cli -h 127.0.0.1 -p 7001 CLUSTER NODES | grep 127.0.0.1:7006
# e.g., output: d4e5f6a1b2c3... 127.0.0.1:7006@17006 slave ...

TARGET_ID="d4e5f6a1b2c3..."

# Step 2: Run CLUSTER FORGET on every OTHER node in the cluster
# (you cannot forget yourself)
for port in 7001 7002 7003 7004 7005; do
  redis-cli -h 127.0.0.1 -p $port CLUSTER FORGET $TARGET_ID
done

# Step 3: Shut down the target node
redis-cli -h 127.0.0.1 -p 7006 SHUTDOWN NOSAVE
```

## Removing a Replica vs a Master

For replicas (nodes with no slots), you can forget them directly without slot migration.

For masters that own slots, you must first migrate all slots to other masters before forgetting:

```bash
# Migrate slots from master at port 7003 before forgetting it
redis-cli --cluster reshard 127.0.0.1:7001 \
  --cluster-from <master-id> \
  --cluster-to <target-master-id> \
  --cluster-slots 5461 \
  --cluster-yes

# Then forget the emptied master
for port in 7001 7002 7004 7005 7006; do
  redis-cli -h 127.0.0.1 -p $port CLUSTER FORGET <master-id>
done
```

## Using redis-cli --cluster del-node

The `redis-cli --cluster del-node` wrapper automates the CLUSTER FORGET process across all nodes:

```bash
redis-cli --cluster del-node 127.0.0.1:7001 <node-id>
```

This command connects to the cluster through the specified entry point and runs `CLUSTER FORGET` on all nodes automatically.

## Verifying a Node Is Forgotten

```bash
# After forgetting, the node should not appear in CLUSTER NODES
redis-cli -h 127.0.0.1 -p 7001 CLUSTER NODES | grep <node-id>
# No output means successfully forgotten
```

## Common Errors

### Cannot Forget Myself

```bash
redis-cli -h 127.0.0.1 -p 7006 CLUSTER FORGET <own-node-id>
# ERR I tried hard but I can't forget myself...
```

You cannot run `CLUSTER FORGET` to remove your own node ID.

### Node Still Owns Slots

```bash
redis-cli -h 127.0.0.1 -p 7001 CLUSTER FORGET <master-with-slots-id>
# ERR Node <id> is not empty! Reshard data away and try again.
```

Migrate all slots before attempting to forget a master node.

## Python Example

```python
import redis

def forget_node_from_all(cluster_nodes, target_node_id):
    """Remove a node from all other cluster members."""
    for host, port in cluster_nodes:
        try:
            r = redis.Redis(host=host, port=port, decode_responses=True)
            r.cluster('forget', target_node_id)
            print(f"Forgot {target_node_id} from {host}:{port}")
        except redis.ResponseError as e:
            print(f"Skipped {host}:{port}: {e}")

cluster_nodes = [
    ('127.0.0.1', 7001),
    ('127.0.0.1', 7002),
    ('127.0.0.1', 7003),
    ('127.0.0.1', 7004),
    ('127.0.0.1', 7005),
]

forget_node_from_all(cluster_nodes, 'd4e5f6a1b2c3...')
```

## Summary

`CLUSTER FORGET` is the command used to remove a node from the cluster topology view. It must be run on every cluster member within 60 seconds to prevent the target node from reappearing via gossip propagation. For replicas, forget them directly; for masters, migrate their slots first. In practice, the `redis-cli --cluster del-node` wrapper is the easiest way to automate the multi-node CLUSTER FORGET process.
