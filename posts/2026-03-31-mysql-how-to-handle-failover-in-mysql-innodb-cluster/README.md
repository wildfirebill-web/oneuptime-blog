# How to Handle Failover in MySQL InnoDB Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB Cluster, Failover, High Availability

Description: Learn how MySQL InnoDB Cluster handles automatic failover, how to trigger manual failover, and how to recover a cluster after a primary failure.

---

## How Automatic Failover Works

MySQL InnoDB Cluster uses Group Replication to detect when the primary becomes unavailable. The remaining members hold an election and promote one of the secondaries to primary. MySQL Router detects the topology change within seconds and redirects new connections to the new primary.

No manual intervention is required for automatic failover when:
- A majority of cluster members are online (quorum is maintained)
- `mysqlrouter` is running and connected to the cluster metadata

## Checking Cluster Health

```javascript
// In MySQL Shell
var cluster = dba.getCluster()
cluster.status()
```

```text
"status": "OK",
"statusText": "Cluster is ONLINE and can tolerate up to ONE failure."
```

If the primary goes down, the status changes temporarily:

```text
"status": "OK_NO_TOLERANCE",
"statusText": "Cluster is NOT tolerant to any failures."
```

After election completes:

```text
"primary": "node2:3306"
```

## Simulating a Failover (Testing)

You can stop the primary to test failover:

```bash
# On node1 (the primary)
sudo systemctl stop mysql
```

Then check from another node:

```javascript
var cluster = dba.getCluster('myCluster')
cluster.status()
```

A new primary should have been elected within seconds.

## Manual Failover (Switchover)

To deliberately move the primary to a specific node (for maintenance):

```javascript
cluster.setPrimaryInstance('clusteradmin@node2:3306')
```

This performs a controlled switchover - the current primary finishes pending transactions before stepping down.

## Recovering a Failed Node

When the failed node comes back online, it can rejoin the cluster:

```bash
# Start MySQL on the failed node
sudo systemctl start mysql
```

```javascript
// From any healthy node in MySQL Shell
var cluster = dba.getCluster()
cluster.rejoinInstance('clusteradmin@node1:3306')
```

MySQL Shell will resync the instance (via clone or incremental recovery) and add it back as a secondary.

## Recovering from Loss of Quorum

If more than half the nodes fail simultaneously, the cluster loses quorum and cannot elect a primary. To restore:

```javascript
// Connect to the most up-to-date surviving node
cluster.forceQuorumUsingPartitionOf('clusteradmin@node1:3306')
```

This forces quorum using the surviving partition. After running this command, add the other nodes back:

```javascript
cluster.rejoinInstance('clusteradmin@node2:3306')
```

## Recovering from a Complete Cluster Outage

If all nodes lose power simultaneously:

```javascript
// On the node with the most recent data
dba.rebootClusterFromCompleteOutage('myCluster')
```

MySQL Shell identifies the most advanced node and makes it the new primary. Other nodes are re-added automatically.

## Preventing Split-Brain

InnoDB Cluster requires a majority (quorum) to process writes. With 3 nodes:
- 2 nodes must be online to accept writes
- If only 1 node remains, the cluster becomes read-only until quorum is restored

This prevents split-brain by design. Maintain an odd number of nodes for optimal fault tolerance.

## Summary

MySQL InnoDB Cluster provides automatic failover through Group Replication, with MySQL Router transparently routing traffic to the new primary. Use `cluster.setPrimaryInstance()` for planned switchovers, `cluster.rejoinInstance()` to re-add recovered nodes, and `cluster.forceQuorumUsingPartitionOf()` to recover from quorum loss.
