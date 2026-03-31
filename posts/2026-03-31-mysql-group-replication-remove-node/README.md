# How to Remove a Node from MySQL Group Replication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Group Replication, Cluster, Node, Maintenance

Description: Learn how to safely remove a node from a MySQL Group Replication cluster, whether for maintenance, decommissioning, or reducing cluster size.

---

## Why You Might Remove a Node

Common reasons to remove a node from group replication include:

- Planned server maintenance or upgrade
- Decommissioning hardware
- Reducing the cluster from 5 nodes to 3
- Replacing a node with a different server

## Graceful Removal Using STOP GROUP_REPLICATION

The cleanest way to remove a node is to stop group replication on it:

```sql
-- Run on the node being removed
STOP GROUP_REPLICATION;
```

The node leaves the group and the remaining members recalculate quorum. For a 3-node cluster, you should only stop a node after confirming the other two are `ONLINE`.

## Verify the Node Left the Group

On any remaining node:

```sql
SELECT MEMBER_HOST, MEMBER_ROLE, MEMBER_STATE
FROM performance_schema.replication_group_members;
```

```text
+---------+-----------+--------+
| node1   | PRIMARY   | ONLINE |
| node2   | SECONDARY | ONLINE |
+---------+-----------+--------+
```

The removed node no longer appears in the list.

## Remove Seed Reference from Remaining Nodes

If the removed node was listed in `group_replication_group_seeds`, update the remaining nodes to remove the reference. This is important so that when any node restarts, it does not attempt to connect to the missing node:

```sql
-- Run on each remaining node
SET GLOBAL group_replication_group_seeds = 'node1:33061,node2:33061';
```

To persist across restarts, update `my.cnf`:

```ini
group_replication_group_seeds = "node1:33061,node2:33061"
```

## Handling an Unresponsive Node

If a node crashes or becomes unreachable, the group detects the absence via the failure detection mechanism. After the `group_replication_member_expel_timeout` period (default 5 seconds), the node is expelled:

```sql
-- Check current expel timeout
SHOW VARIABLES LIKE 'group_replication_member_expel_timeout';
```

```text
+----------------------------------------+-------+
| Variable_name                          | Value |
+----------------------------------------+-------+
| group_replication_member_expel_timeout | 5     |
+----------------------------------------+-------+
```

You can increase this for unreliable networks:

```sql
SET GLOBAL group_replication_member_expel_timeout = 10;
```

## Force Remove a Node via allowlist

If a failed node keeps trying to rejoin and disrupting the cluster, explicitly block it using the `group_replication_ip_allowlist` variable:

```sql
-- On remaining nodes, restrict to known good nodes
SET GLOBAL group_replication_ip_allowlist = 'node1,node2,node3';
```

## Primary Election After Removing the Primary

If you remove the current primary node, the group automatically elects a new primary. Check which node became primary:

```sql
SELECT MEMBER_HOST, MEMBER_ROLE
FROM performance_schema.replication_group_members
WHERE MEMBER_ROLE = 'PRIMARY';
```

You can also manually switch the primary before removal:

```sql
-- Promote node2 to primary before stopping node1
SELECT group_replication_set_as_primary('uuid-of-node2');
```

Get the UUID from:

```sql
SELECT MEMBER_ID, MEMBER_HOST
FROM performance_schema.replication_group_members;
```

## Summary

Removing a node from MySQL Group Replication is done by running `STOP GROUP_REPLICATION` on the target node. Update `group_replication_group_seeds` on remaining nodes to exclude the removed member. If the primary is being removed, either manually trigger election first with `group_replication_set_as_primary()` or let the automatic election handle it. Unresponsive nodes are automatically expelled after the `group_replication_member_expel_timeout` period.
