# What Is MySQL InnoDB ReplicaSet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB ReplicaSet, Replication, MySQL Shell, High Availability

Description: MySQL InnoDB ReplicaSet is a MySQL Shell managed source-replica topology providing simplified management without Group Replication's consensus overhead.

---

## Overview

MySQL InnoDB ReplicaSet is a lighter-weight alternative to InnoDB Cluster. It uses traditional asynchronous or semi-synchronous source-replica replication (not Group Replication) but wraps it with MySQL Shell's `dba` management API for simplified provisioning, monitoring, and failover. It is suited for deployments that need easier management without the complexity or quorum requirements of Group Replication.

## How It Differs from InnoDB Cluster

| Feature | InnoDB ReplicaSet | InnoDB Cluster |
|---|---|---|
| Replication type | Async/semi-sync | Group Replication |
| Automatic failover | Manual (via Shell) | Automatic |
| Quorum requirement | No | Yes (majority needed) |
| Conflict detection | No | Yes |
| Minimum instances | 1 source + 1 replica | 3 |

Because InnoDB ReplicaSet does not use Group Replication, it does not require a quorum to operate. A single source with one or more replicas is sufficient.

## Creating an InnoDB ReplicaSet

Use MySQL Shell:

```javascript
// Connect to the source instance
shell.connect('admin@source-host:3306')

// Create the ReplicaSet
var rs = dba.createReplicaSet('myReplicaSet')

// Add a replica
rs.addInstance('admin@replica-host:3306')

// Check status
rs.status()
```

The `status()` output shows the topology:

```json
{
  "replicaSet": {
    "name": "myReplicaSet",
    "topology": {
      "source-host:3306": {"address": "source-host:3306", "mode": "R/W", "status": "ONLINE"},
      "replica-host:3306": {"address": "replica-host:3306", "mode": "R/O", "status": "ONLINE"}
    }
  }
}
```

## Performing a Planned Failover

When you need to promote a replica to source (e.g., for maintenance):

```javascript
var rs = dba.getReplicaSet()

// Graceful switchover: source must be reachable
rs.setPrimaryInstance('admin@replica-host:3306')
```

This ensures the replica is fully caught up before the switchover.

## Performing an Emergency Failover

If the source is lost:

```javascript
// Force failover to the least-behind replica
rs.forcePrimaryInstance('admin@replica-host:3306')
```

Note: `forcePrimaryInstance` may result in data loss if the new primary had not applied all transactions from the failed source. MySQL Shell will warn you about the risk.

## Integrating with MySQL Router

InnoDB ReplicaSet also works with MySQL Router for transparent connection routing:

```bash
mysqlrouter --bootstrap admin@source-host:3306 --directory /var/lib/mysqlrouter
```

Router routes writes to the primary and reads to secondaries, just as with InnoDB Cluster.

## Adding Read Replicas

You can add additional replicas to distribute read traffic:

```javascript
rs.addInstance('admin@replica2-host:3306')
rs.addInstance('admin@replica3-host:3306')
```

## Summary

MySQL InnoDB ReplicaSet provides a MySQL Shell managed wrapper around traditional source-replica replication. It offers simplified setup, status monitoring, and switchover/failover commands without requiring Group Replication's quorum or consensus protocol. It is a good choice for smaller deployments or environments where the overhead of Group Replication is not justified but consistent tooling for HA management is desired.
