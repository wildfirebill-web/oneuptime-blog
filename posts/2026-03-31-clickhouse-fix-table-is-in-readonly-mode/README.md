# How to Fix "Table is in readonly mode" in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Readonly, ReplicatedMergeTree, ZooKeeper, Troubleshooting

Description: Fix "Table is in readonly mode" errors in ClickHouse ReplicatedMergeTree tables caused by ZooKeeper connectivity or metadata issues.

---

The "Table is in readonly mode" error occurs exclusively with `ReplicatedMergeTree` family tables. ClickHouse sets a table to readonly when it cannot communicate with ZooKeeper (or ClickHouse Keeper) or detects a metadata inconsistency. In readonly mode, the table accepts SELECT queries but rejects INSERT and DDL operations.

## Identify Affected Tables

```sql
SELECT
    database,
    name,
    is_readonly,
    zookeeper_exception
FROM system.replicas
WHERE is_readonly = 1;
```

## Check ZooKeeper Connectivity

```bash
# Test connection to ZooKeeper from the ClickHouse host
echo ruok | nc zookeeper-host 2181
# Expected response: imok

# Or check ClickHouse's ZooKeeper status
clickhouse-client --query "SELECT * FROM system.zookeeper WHERE path = '/'" 2>&1 | head -5
```

If ZooKeeper is unreachable, ClickHouse puts replicated tables into readonly mode automatically.

## Fix ZooKeeper Connection Issues

Check `config.xml` for the correct ZooKeeper address:

```xml
<zookeeper>
  <node>
    <host>zookeeper-host</host>
    <port>2181</port>
  </node>
</zookeeper>
```

Verify ZooKeeper is healthy:

```bash
echo stat | nc zookeeper-host 2181 | head -20
```

## Re-Initialize a Table Stuck in Readonly

If ZooKeeper is healthy but the table remains readonly, restart the server to force re-initialization:

```bash
sudo systemctl restart clickhouse-server
```

Check the error after restart:

```sql
SELECT database, name, zookeeper_exception
FROM system.replicas
WHERE is_readonly = 1;
```

## Restore Table from ZooKeeper Metadata Loss

If the ZooKeeper path for the table was deleted, detach and re-attach:

```sql
DETACH TABLE my_database.my_table;
-- Fix ZooKeeper path or recreate it
ATTACH TABLE my_database.my_table;
```

Or drop and recreate the ZooKeeper path explicitly:

```sql
SYSTEM RESTORE REPLICA my_database.my_table;
```

## Check Disk Space

A full disk also triggers readonly mode:

```bash
df -h /var/lib/clickhouse
```

Free up space or expand the disk, then restart ClickHouse.

## Summary

"Table is in readonly mode" in ClickHouse means a ReplicatedMergeTree table lost its ZooKeeper connection or encountered a metadata error. Check ZooKeeper connectivity, verify `config.xml` settings, inspect `system.replicas` for the exception message, and restart the server after resolving the underlying issue. Use `SYSTEM RESTORE REPLICA` if ZooKeeper metadata needs reconstruction.
