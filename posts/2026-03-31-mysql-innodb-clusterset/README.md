# What Is MySQL InnoDB ClusterSet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB ClusterSet, High Availability, Disaster Recovery, Replication

Description: MySQL InnoDB ClusterSet links multiple InnoDB Clusters across data centers, enabling cross-site disaster recovery with automated failover between clusters.

---

## Overview

MySQL InnoDB ClusterSet is a deployment topology that extends InnoDB Cluster to span multiple data centers or availability zones. It connects a primary InnoDB Cluster with one or more replica InnoDB Clusters through asynchronous replication. If the primary cluster becomes unavailable due to a site failure, the ClusterSet can promote a replica cluster to become the new primary, restoring service across the entire deployment.

ClusterSet was introduced in MySQL 8.0.27 and is managed through MySQL Shell, the same tool used to administer InnoDB Clusters.

## Architecture

A ClusterSet consists of:
- One **primary cluster** - receives all write traffic, replicates to replica clusters
- One or more **replica clusters** - receive replicated data, serve read traffic locally, can be promoted on failure
- **MySQL Router** - routes application traffic to the correct cluster based on ClusterSet state

```text
Data Center A:                    Data Center B:
  Primary InnoDB Cluster            Replica InnoDB Cluster
  (3 nodes, Group Replication)  --> (3 nodes, Group Replication)
  Primary: node1                    All read-only by default

  Async replication channel between clusters
```

## Creating a ClusterSet with MySQL Shell

```javascript
// Connect to the primary cluster's primary instance
var cluster = dba.getCluster('PrimaryCluster');

// Create a ClusterSet from the existing cluster
var clusterSet = cluster.createClusterSet('myClusterSet');

// Create a replica cluster at a second site
// (Connect MySQL Shell to a server at the DR site first)
var replicaCluster = clusterSet.createReplicaCluster(
  'mysql://admin@dr-site-host:3306',
  'ReplicaCluster'
);

// Check the ClusterSet status
clusterSet.status();
```

```javascript
// Expected status output structure:
{
  "clusterSetName": "myClusterSet",
  "primaryCluster": "PrimaryCluster",
  "status": "HEALTHY",
  "clusters": {
    "PrimaryCluster": {
      "clusterRole": "PRIMARY",
      "globalStatus": "OK"
    },
    "ReplicaCluster": {
      "clusterRole": "REPLICA",
      "globalStatus": "OK",
      "clusterSetReplicationStatus": "OK"
    }
  }
}
```

## Monitoring Replication Between Clusters

```javascript
// Check detailed ClusterSet status
clusterSet.status({extended: 1});

// Check replication lag between primary and replica clusters
clusterSet.status({extended: 2});
```

```sql
-- Check the async replication channel used by ClusterSet
SELECT
  CHANNEL_NAME,
  SERVICE_STATE,
  LAST_ERROR_MESSAGE
FROM performance_schema.replication_connection_status
WHERE CHANNEL_NAME LIKE 'clusterset_%';
```

## Controlled Switchover

A controlled switchover promotes a replica cluster to primary without data loss. Use this for planned maintenance on the primary site:

```javascript
// Promote ReplicaCluster to primary (graceful, no data loss)
clusterSet.setPrimaryCluster('ReplicaCluster');

// The old primary cluster becomes a replica automatically
clusterSet.status();
```

## Emergency Failover

An emergency failover is used when the primary cluster is unreachable:

```javascript
// Force failover to ReplicaCluster (may lose recently committed transactions)
clusterSet.forcePrimaryCluster('ReplicaCluster');

// After the failed site recovers, rejoin it as a replica
clusterSet.rejoinCluster('PrimaryCluster');
```

## MySQL Router Configuration for ClusterSet

MySQL Router must be bootstrapped against the ClusterSet to automatically route traffic to the current primary cluster:

```bash
mysqlrouter --bootstrap admin@primary-cluster-host:3306 \
  --account router_user \
  --directory /etc/mysqlrouter \
  --conf-use-gr-notifications
```

Router reads ClusterSet metadata and updates its routing rules whenever the primary cluster changes, providing transparent failover for applications.

## Summary

MySQL InnoDB ClusterSet connects multiple InnoDB Clusters across geographically distributed sites through asynchronous replication, enabling cross-site disaster recovery. The primary cluster receives writes and replicates to replica clusters. On primary site failure, a replica cluster can be promoted to primary through an emergency failover. For planned maintenance, a controlled switchover promotes a replica with no data loss. MySQL Router automatically routes application traffic to the current primary cluster. ClusterSet is managed entirely through MySQL Shell and is the recommended approach for multi-site MySQL high availability deployments.
