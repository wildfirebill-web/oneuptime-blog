# How to Fix "Table is in readonly mode" in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Readonly Mode, Troubleshooting, ZooKeeper, Replication

Description: Fix ClickHouse "Table is in readonly mode" errors caused by ZooKeeper connectivity issues, replication errors, or disk space problems in replicated table setups.

---

## Overview

The error "Table is in readonly mode" in ClickHouse occurs when a replicated table (using `ReplicatedMergeTree` engines) cannot communicate with ZooKeeper (or ClickHouse Keeper). When ZooKeeper is unavailable or the replica detects inconsistencies, it switches to read-only mode to prevent data corruption. The table accepts `SELECT` queries but rejects any writes.

## Identify Affected Tables

```sql
-- Check which tables are in readonly mode
SELECT database, table, is_readonly, zookeeper_path
FROM system.replicas
WHERE is_readonly = 1;

-- Detailed status
SELECT
    database,
    table,
    is_readonly,
    is_session_expired,
    future_parts,
    queue_size,
    last_exception
FROM system.replicas
WHERE is_readonly = 1;
```

## Check ZooKeeper Connectivity

```sql
-- Check if ZooKeeper is reachable
SELECT * FROM system.zookeeper WHERE path = '/';

-- Check ZooKeeper session status
SELECT * FROM system.zookeeper_connection;
```

If ZooKeeper is unreachable, fix the ZooKeeper cluster first.

## Check ZooKeeper Configuration

```xml
<!-- /etc/clickhouse-server/config.xml -->
<zookeeper>
    <node>
        <host>zk1.internal</host>
        <port>2181</port>
    </node>
    <node>
        <host>zk2.internal</host>
        <port>2181</port>
    </node>
    <node>
        <host>zk3.internal</host>
        <port>2181</port>
    </node>
    <session_timeout_ms>30000</session_timeout_ms>
    <operation_timeout_ms>10000</operation_timeout_ms>
</zookeeper>
```

Test ZooKeeper connectivity from the ClickHouse server:

```bash
# Install zookeeper client tools
# Test connection
echo ruok | nc zk1.internal 2181
echo stat | nc zk1.internal 2181
```

## Restart the Replica

Once ZooKeeper is available again, restart the replica replication:

```sql
SYSTEM RESTART REPLICA database.table_name;

-- Or restart all replicas in a database
SYSTEM RESTART REPLICAS;
```

## Check Disk Space

Readonly mode can also trigger if the data disk is full:

```bash
df -h /var/lib/clickhouse
```

If the disk is full, free space first:

```bash
# Check largest directories
du -sh /var/lib/clickhouse/data/*
du -sh /var/lib/clickhouse/tmp/*

# Clear temp files
sudo rm -rf /var/lib/clickhouse/tmp/*
```

Then remove old partitions in ClickHouse:

```sql
-- List partitions
SELECT partition, rows, bytes_on_disk / 1073741824 AS size_gb
FROM system.parts
WHERE database = 'mydb' AND table = 'events' AND active = 1
ORDER BY partition;

-- Drop old partitions
ALTER TABLE mydb.events DROP PARTITION '2023-01';
```

## Check for Replication Errors

```sql
-- View replication queue errors
SELECT
    database,
    table,
    type,
    last_exception,
    last_exception_time
FROM system.replication_queue
WHERE last_exception != ''
ORDER BY last_exception_time DESC
LIMIT 20;

-- Check replica status
SELECT *
FROM system.replicas
WHERE last_exception != '';
```

## Sync Replica with ZooKeeper

If the replica is out of sync:

```sql
-- Force replica to sync with ZooKeeper state
SYSTEM SYNC REPLICA database.table_name;
```

## Detach and Reattach (Last Resort)

If a table is stuck in readonly and cannot recover:

```sql
-- Detach the table
DETACH TABLE mydb.broken_table;

-- Reattach
ATTACH TABLE mydb.broken_table;

-- Or drop and recreate with ATTACH existing data
```

## Non-Replicated Tables in Readonly Mode

For non-replicated tables, readonly mode is caused by the `readonly` user setting or disk issues:

```sql
-- Check user setting
SELECT value FROM system.settings WHERE name = 'readonly';

-- If readonly = 1, change for session
SET readonly = 0;
```

## Summary

"Table is in readonly mode" in ClickHouse primarily occurs with `ReplicatedMergeTree` tables when ZooKeeper is unavailable, a ZooKeeper session expires, or the data disk is full. Diagnose with `system.replicas` to identify affected tables and `last_exception` to see the root cause. Fix by restoring ZooKeeper connectivity, running `SYSTEM RESTART REPLICA`, or freeing disk space. Monitor replication health proactively via `system.replicas` and `system.replication_queue`.
