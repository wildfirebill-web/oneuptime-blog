# How to Add a Node to a MySQL Group Replication Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Group Replication, Cluster, Node, High Availability

Description: Step-by-step guide to adding a new member node to a running MySQL Group Replication cluster with minimal impact to existing operations.

---

## Overview

Adding a node to an existing MySQL Group Replication cluster involves configuring the new server, creating the recovery user, and starting group replication. The new node uses a distributed recovery mechanism to catch up with existing members automatically before being marked as ONLINE.

## Prepare the New Node

Install MySQL 8.0 and configure `/etc/mysql/mysql.conf.d/mysqld.cnf` with group replication settings that match the existing cluster:

```ini
[mysqld]
server_id = 4
gtid_mode = ON
enforce_gtid_consistency = ON
log_bin = mysql-bin
binlog_format = ROW
log_replica_updates = ON
plugin_load_add = group_replication.so

group_replication_group_name = "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
group_replication_start_on_boot = OFF
group_replication_local_address = "node4:33061"
group_replication_group_seeds = "node1:33061,node2:33061,node3:33061"
group_replication_bootstrap_group = OFF
group_replication_single_primary_mode = ON
group_replication_enforce_update_everywhere_checks = OFF
```

Use a unique `server_id` that is not in use by any existing member. The `group_replication_group_name` must exactly match the existing cluster's UUID.

## Create the Recovery User on the New Node

Connect to the new node and create the replication user:

```sql
SET SQL_LOG_BIN = 0;
CREATE USER 'repl'@'%' IDENTIFIED BY 'replPassword1!' REQUIRE SSL;
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN = 1;
```

Wrapping the user creation in `SET SQL_LOG_BIN = 0` prevents it from being replicated as a binary log event during the recovery phase.

## Configure the Recovery Channel

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_USER = 'repl',
  SOURCE_PASSWORD = 'replPassword1!'
  FOR CHANNEL 'group_replication_recovery';
```

## Start Group Replication on the New Node

```sql
START GROUP_REPLICATION;
```

The node will go through a `RECOVERING` state while it catches up with the existing group. Depending on how much data exists, this may take a few seconds or several minutes.

## Monitor the Join Progress

On any existing node, watch the member states:

```sql
SELECT MEMBER_HOST, MEMBER_ROLE, MEMBER_STATE
FROM performance_schema.replication_group_members;
```

```text
+---------+-----------+-----------+
| node1   | PRIMARY   | ONLINE    |
| node2   | SECONDARY | ONLINE    |
| node3   | SECONDARY | ONLINE    |
| node4   | SECONDARY | RECOVERING|
+---------+-----------+-----------+
```

Wait until node4 shows `ONLINE` before directing any traffic to it.

## Check the Recovery Progress

```sql
SELECT
  CHANNEL_NAME,
  SERVICE_STATE,
  COUNT_RECEIVED_HEARTBEATS
FROM performance_schema.replication_connection_status
WHERE CHANNEL_NAME = 'group_replication_recovery';
```

## Verify Network Connectivity Before Adding

Port 33061 must be open between all nodes. Test with:

```bash
# From node4, test connectivity to existing nodes
nc -zv node1 33061
nc -zv node2 33061
nc -zv node3 33061
```

## Firewall Rules (Linux)

```bash
# Allow group replication port
sudo firewall-cmd --permanent --add-port=33061/tcp
sudo firewall-cmd --reload
```

## Summary

To add a node to a MySQL Group Replication cluster, configure `my.cnf` with matching group settings, create the recovery user, configure the recovery channel, and run `START GROUP_REPLICATION`. The node automatically recovers using distributed recovery and transitions to `ONLINE` once it has synchronized with the group. Monitor progress using `performance_schema.replication_group_members`.
