# How to Set Up MySQL Group Replication in Multi-Primary Mode

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Group Replication, Multi-Primary, High Availability, Replication

Description: Configure MySQL Group Replication in multi-primary mode so every node accepts writes, enabling horizontal write scaling with automatic conflict detection.

---

## What Is Multi-Primary Mode

In multi-primary mode, all nodes in the group act as primaries and accept write operations simultaneously. MySQL uses optimistic concurrency control - transactions are certified against concurrent transactions on other nodes before committing. If a conflict is detected, one transaction is rolled back.

Multi-primary mode is appropriate when you have geographically distributed applications that need to write to a local database node.

## Prerequisites

- MySQL 8.0+ on all nodes
- GTIDs enabled
- Binary logging with `ROW` format
- The `group_replication` plugin installed

## Configure MySQL for Multi-Primary

Edit `my.cnf` on each node. The key difference from single-primary mode is setting `group_replication_single_primary_mode = OFF` and enabling `enforce_update_everywhere_checks`:

```ini
[mysqld]
server_id = 1
gtid_mode = ON
enforce_gtid_consistency = ON
log_bin = mysql-bin
binlog_format = ROW
log_replica_updates = ON
plugin_load_add = group_replication.so

group_replication_group_name = "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
group_replication_start_on_boot = OFF
group_replication_local_address = "node1:33061"
group_replication_group_seeds = "node1:33061,node2:33061,node3:33061"
group_replication_bootstrap_group = OFF

# Multi-primary mode settings
group_replication_single_primary_mode = OFF
group_replication_enforce_update_everywhere_checks = ON
```

## Bootstrap the Group

On node1 only:

```sql
SET GLOBAL group_replication_bootstrap_group = ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group = OFF;
```

## Join Remaining Nodes

On node2 and node3:

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_USER='repl',
  SOURCE_PASSWORD='replPassword1!'
  FOR CHANNEL 'group_replication_recovery';

START GROUP_REPLICATION;
```

## Verify All Nodes Are Primary

```sql
SELECT MEMBER_HOST, MEMBER_ROLE, MEMBER_STATE
FROM performance_schema.replication_group_members;
```

```text
+---------+---------+---------+
| node1   | PRIMARY | ONLINE  |
| node2   | PRIMARY | ONLINE  |
| node3   | PRIMARY | ONLINE  |
+---------+---------+---------+
```

## Handle Write Conflicts

When two nodes write to the same row at the same time, one transaction is rolled back with error 1213 (deadlock). Your application must retry:

```python
import mysql.connector
import time

def execute_with_retry(conn, sql, params, retries=3):
    for attempt in range(retries):
        try:
            cursor = conn.cursor()
            cursor.execute(sql, params)
            conn.commit()
            return
        except mysql.connector.Error as e:
            if e.errno == 1213 and attempt < retries - 1:
                time.sleep(0.1 * (attempt + 1))
                continue
            raise
```

## Switch from Single-Primary to Multi-Primary Online

You can switch modes without restarting the cluster using UDFs:

```sql
-- Run on any group member
SELECT group_replication_switch_to_multi_primary_mode();
```

To switch back to single-primary:

```sql
SELECT group_replication_switch_to_single_primary_mode();
```

## Performance Considerations

Multi-primary mode introduces certification overhead for every transaction. Tables without primary keys are not allowed because conflicts cannot be detected reliably. Enforce this with:

```sql
-- Check for tables without primary keys
SELECT table_schema, table_name
FROM information_schema.tables t
LEFT JOIN information_schema.table_constraints c
  ON t.table_schema = c.table_schema
  AND t.table_name = c.table_name
  AND c.constraint_type = 'PRIMARY KEY'
WHERE c.constraint_name IS NULL
  AND t.table_type = 'BASE TABLE'
  AND t.table_schema NOT IN ('mysql', 'sys', 'information_schema', 'performance_schema');
```

## Summary

MySQL Group Replication in multi-primary mode enables all nodes to accept writes simultaneously. Configure it by disabling `group_replication_single_primary_mode` and enabling `enforce_update_everywhere_checks`. Applications must handle transaction rollbacks from write conflicts. Use the `group_replication_switch_to_multi_primary_mode()` UDF to switch modes online without restarting the cluster.
