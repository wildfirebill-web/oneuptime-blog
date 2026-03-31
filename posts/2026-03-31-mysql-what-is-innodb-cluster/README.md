# What Is MySQL InnoDB Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB Cluster, High Availability, Group Replication, MySQL Shell

Description: MySQL InnoDB Cluster is a complete high-availability solution combining Group Replication, MySQL Router, and MySQL Shell for automatic failover and read scaling.

---

## Overview

MySQL InnoDB Cluster is an integrated high-availability solution that combines three components into a cohesive system:

1. **MySQL Group Replication**: Provides distributed, fault-tolerant replication with automatic failover.
2. **MySQL Router**: Transparently routes application connections to the appropriate cluster member (primary for writes, secondaries for reads).
3. **MySQL Shell**: Provides the `dba` administration API to create, manage, and monitor the cluster.

Together, these components provide a production-ready HA solution without requiring external tooling like Orchestrator or MHA.

## Architecture

A typical InnoDB Cluster consists of a minimum of three MySQL instances arranged in a Group Replication group. MySQL Router runs alongside the application and maintains awareness of which member is currently the primary:

```
Application -> MySQL Router -> [Primary | Secondary | Secondary]
```

When the primary fails, Group Replication elects a new primary automatically. MySQL Router detects the change within seconds and redirects write connections to the new primary without application reconfiguration.

## Creating a Cluster with MySQL Shell

```javascript
// Connect to the first MySQL instance
shell.connect('root@node1:3306')

// Create the cluster
var cluster = dba.createCluster('myCluster')

// Add more instances
cluster.addInstance('root@node2:3306')
cluster.addInstance('root@node3:3306')

// Check cluster status
cluster.status()
```

The `status()` output shows each member's role and state:

```json
{
  "clusterName": "myCluster",
  "defaultReplicaSet": {
    "topology": {
      "node1:3306": {"mode": "R/W", "status": "ONLINE"},
      "node2:3306": {"mode": "R/O", "status": "ONLINE"},
      "node3:3306": {"mode": "R/O", "status": "ONLINE"}
    }
  }
}
```

## Configuring MySQL Router

After creating the cluster, bootstrap MySQL Router:

```bash
mysqlrouter --bootstrap root@node1:3306 --directory /var/lib/mysqlrouter
mysqlrouter --config /var/lib/mysqlrouter/mysqlrouter.conf &
```

Router creates four ports by default:
- `6446`: Read/Write (routes to primary)
- `6447`: Read-Only (routes to secondaries, round-robin)
- `64460`: X Protocol Read/Write
- `64470`: X Protocol Read-Only

Applications connect to the router ports rather than directly to MySQL instances.

## Handling Failover

When a primary instance fails, the cluster automatically elects a new primary. You can observe this:

```javascript
// Check current primary
cluster.status()

// After a failure, forcefully rejoin a recovered instance
cluster.rejoinInstance('root@node1:3306')

// Recover from complete outage
dba.rebootClusterFromCompleteOutage()
```

## Read Scale-Out

In single-primary mode, all secondaries accept read traffic. Router distributes read connections across them automatically. For applications that can tolerate slightly stale reads, this reduces load on the primary:

```python
import mysql.connector
# Connect to read-only port for SELECT workloads
conn = mysql.connector.connect(host='router-host', port=6447, ...)
```

## Summary

MySQL InnoDB Cluster integrates Group Replication, MySQL Router, and MySQL Shell into a complete HA solution. It provides automatic failover, transparent connection routing, and a simple management API. It is the recommended approach for production MySQL deployments requiring high availability without complex external tooling.
