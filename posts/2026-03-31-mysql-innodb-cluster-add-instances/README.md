# How to Add Instances to a MySQL InnoDB Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB Cluster, MySQL Shell, High Availability, Cluster

Description: Use MySQL Shell to add new instances to an existing InnoDB Cluster, with automatic provisioning and state recovery for seamless cluster expansion.

---

## Prerequisites

Before adding an instance to an InnoDB Cluster, ensure:

- MySQL 8.0+ is installed on the new instance
- The instance is reachable on port 3306 and 33061 from all cluster members
- The instance has not previously been part of a cluster (clean state)

## Check Instance Compatibility

Connect to MySQL Shell and run a pre-check:

```javascript
// In MySQL Shell (mysqlsh)
shell.connect('admin@node1:3306')
var cluster = dba.getCluster()
dba.checkInstanceConfiguration('admin@node4:3306')
```

If the instance needs configuration changes, MySQL Shell will suggest them:

```text
Please use the dba.configureInstance() command to repair these issues.
```

## Configure the New Instance

```javascript
dba.configureInstance('admin@node4:3306')
```

MySQL Shell will prompt for the admin password and apply required settings such as GTID mode, binary logging, and performance schema configuration.

After configuration, restart MySQL on the new instance:

```bash
sudo systemctl restart mysql
```

## Add the Instance to the Cluster

Reconnect to Shell and add the new instance:

```javascript
shell.connect('admin@node1:3306')
var cluster = dba.getCluster()
cluster.addInstance('admin@node4:3306')
```

MySQL Shell will ask how to provision the new instance:

```text
Please select a recovery method [C]lone/[I]ncremental recovery/[A]bort (default Clone):
```

- **Clone** - uses MySQL Clone plugin to copy the dataset (recommended for large datasets)
- **Incremental recovery** - replays binary logs (faster for recently added instances)

## Monitor the Recovery Progress

```javascript
cluster.status()
```

```text
{
    "clusterName": "myCluster",
    "status": "OK",
    "topology": {
        "node1:3306": {"status": "ONLINE", "role": "HA"},
        "node2:3306": {"status": "ONLINE", "role": "HA"},
        "node3:3306": {"status": "ONLINE", "role": "HA"},
        "node4:3306": {"status": "RECOVERING", "role": "HA"}
    }
}
```

Wait for node4 to show `"status": "ONLINE"` before directing traffic to it.

## Verify the Instance Was Added

```javascript
cluster.status({extended: true})
```

Check the `groupReplicationMembers` section to confirm all expected members are listed.

## Add Instance with Custom Options

You can customize the addition with options:

```javascript
cluster.addInstance('admin@node4:3306', {
  recoveryMethod: 'clone',
  label: 'node4-replica',
  waitRecovery: 2  // 0=no wait, 1=wait but no output, 2=wait with progress
})
```

## Update MySQL Router After Adding an Instance

If MySQL Router was bootstrapped against the cluster, restart it to pick up the new member:

```bash
sudo systemctl restart mysqlrouter
```

Or re-bootstrap:

```bash
mysqlrouter --bootstrap admin@node1:3306 --user=mysqlrouter --force
```

## Summary

Add instances to a MySQL InnoDB Cluster using `cluster.addInstance()` in MySQL Shell. Run `dba.checkInstanceConfiguration()` and `dba.configureInstance()` first to ensure the new server meets requirements. Choose between Clone (for large datasets) or incremental recovery (for small catch-up). Monitor progress with `cluster.status()` and restart MySQL Router to include the new instance in routing.
