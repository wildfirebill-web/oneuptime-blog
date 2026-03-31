# How to Use MySQL Shell for InnoDB Cluster Administration

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB, Cluster, Shell, High Availability

Description: Learn how to use MySQL Shell's dba object to create, manage, and monitor InnoDB Clusters for high availability and automatic failover.

---

## Introduction

MySQL InnoDB Cluster provides a complete high-availability solution built on MySQL Group Replication. MySQL Shell is the primary tool for administering InnoDB Clusters, offering a `dba` global object with methods to create and manage clusters. This guide covers the essential commands for cluster administration.

## Prerequisites

Before creating a cluster, each MySQL instance must be configured for Group Replication. Use the `dba.configureInstance()` method to prepare a node:

```javascript
dba.configureInstance('root@node1:3306')
```

This checks and applies necessary configuration changes (persisted using `SET PERSIST`). Repeat for all nodes.

## Creating an InnoDB Cluster

Connect to the primary node and create the cluster:

```javascript
shell.connect('root@node1:3306')
var cluster = dba.createCluster('myCluster')
```

The output confirms the cluster is created with a single-primary mode by default. Add secondary nodes:

```javascript
cluster.addInstance('root@node2:3306')
cluster.addInstance('root@node3:3306')
```

MySQL Shell will prompt you to choose a recovery method - select `Clone` for a clean data copy, or `Incremental` if the instance already has the dataset.

## Checking Cluster Status

Use `status()` to view the health of all cluster members:

```javascript
cluster.status()
```

Sample output:

```json
{
  "clusterName": "myCluster",
  "defaultReplicaSet": {
    "name": "default",
    "primary": "node1:3306",
    "status": "OK",
    "topology": {
      "node1:3306": {"mode": "R/W", "status": "ONLINE"},
      "node2:3306": {"mode": "R/O", "status": "ONLINE"},
      "node3:3306": {"mode": "R/O", "status": "ONLINE"}
    }
  }
}
```

## Performing a Failover

If the primary goes offline, the cluster elects a new primary automatically. To manually switch the primary, use:

```javascript
cluster.setPrimaryInstance('root@node2:3306')
```

To force an election when quorum is lost:

```javascript
cluster.forceQuorumUsingPartitionOf('root@node2:3306')
```

## Removing an Instance

To gracefully remove a member from the cluster:

```javascript
cluster.removeInstance('root@node3:3306')
```

If the node is unreachable, add the `force` option:

```javascript
cluster.removeInstance('root@node3:3306', {force: true})
```

## Rejoining an Instance

After maintenance or a crash, rejoin a node with:

```javascript
cluster.rejoinInstance('root@node3:3306')
```

## Dissolving a Cluster

To stop and remove the cluster entirely:

```javascript
cluster.dissolve({force: true})
```

This removes Group Replication configuration from all members.

## Getting a Cluster Object After Reconnect

If you disconnect and reconnect, retrieve the existing cluster object:

```javascript
shell.connect('root@node1:3306')
var cluster = dba.getCluster()
cluster.status()
```

## Monitoring with describe()

The `describe()` method gives a structural overview:

```javascript
cluster.describe()
```

This lists all instances with their roles and addresses, useful for quick audits.

## Summary

MySQL Shell's `dba` object provides a complete interface for InnoDB Cluster administration - from initial setup with `configureInstance()` and `createCluster()`, to day-to-day operations like adding or removing nodes, monitoring status, and performing failovers. Mastering these commands is essential for maintaining high-availability MySQL deployments.
