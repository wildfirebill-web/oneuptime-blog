# How to Remove Instances from a MySQL InnoDB Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, InnoDB Cluster, MySQL Shell, Cluster, Maintenance

Description: Learn how to safely remove instances from a MySQL InnoDB Cluster using MySQL Shell, including handling offline instances and updating MySQL Router.

---

## When to Remove an Instance

You may need to remove an instance when:

- Decommissioning a server
- Replacing hardware
- Reducing cluster size from 5 to 3 nodes
- Migrating to a different host

## Remove an Online Instance

Connect to MySQL Shell and remove the instance:

```javascript
// Connect to any online cluster member
shell.connect('admin@node1:3306')
var cluster = dba.getCluster()

// Remove node4 from the cluster
cluster.removeInstance('admin@node4:3306')
```

MySQL Shell will confirm:

```text
The instance will be removed from the InnoDB cluster. Depending on the
instance being the Seed or not, the Metadata cache may need to be updated.

Instance successfully removed from the cluster.
```

## Remove an Offline or Unreachable Instance

If the instance is unreachable, use the `force` option:

```javascript
cluster.removeInstance('admin@node4:3306', {force: true})
```

This removes the instance from cluster metadata even if it cannot be contacted. Ensure the instance does not restart and attempt to rejoin with stale data.

## Verify the Instance Was Removed

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
        "node3:3306": {"status": "ONLINE", "role": "HA"}
    }
}
```

The removed instance no longer appears in the topology.

## Clean Up the Removed Instance

On the removed server, stop group replication to ensure it does not rejoin:

```sql
STOP GROUP_REPLICATION;
SET GLOBAL group_replication_start_on_boot = OFF;
```

To fully reset the instance so it can be reused:

```javascript
// In MySQL Shell connected to the removed instance
dba.resetInstance('admin@node4:3306')
```

This removes all group replication metadata from the instance.

## Handle a Primary Instance Removal

If you are removing the primary instance, first switch the primary to another node:

```javascript
cluster.setPrimaryInstance('admin@node2:3306')
```

Verify the switch:

```javascript
cluster.status()
```

Confirm node2 is now `PRIMARY` before removing node1:

```javascript
cluster.removeInstance('admin@node1:3306')
```

## Update MySQL Router

After removing an instance, restart MySQL Router so it updates its cached topology:

```bash
sudo systemctl restart mysqlrouter
```

Check the router logs to confirm the removed instance is no longer listed:

```bash
sudo journalctl -u mysqlrouter -n 50 | grep "member"
```

## Check Quorum After Removal

Ensure the remaining cluster still has quorum:

```javascript
cluster.status()
// Look for "status": "OK" - not "NO_QUORUM" or "OK_NO_TOLERANCE"
```

For a 3-node cluster, removing one node leaves 2 nodes which can still maintain quorum. Removing a second would leave 1 node with no quorum.

## Summary

Use `cluster.removeInstance()` in MySQL Shell to remove instances from a MySQL InnoDB Cluster. For offline instances, add `{force: true}`. If removing the primary, switch primary first with `cluster.setPrimaryInstance()`. Clean up the removed instance using `dba.resetInstance()` and restart MySQL Router to update routing topology.
