# How to Fix 'Table is in readonly mode' in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Troubleshooting, Replication, ZooKeeper, Readonly

Description: Fix 'Table is in readonly mode' errors in ClickHouse caused by ZooKeeper connectivity issues, stale replicas, or disk space exhaustion.

---

The "Table is in readonly mode" error in ClickHouse means a ReplicatedMergeTree table has been placed in read-only mode and will refuse INSERT, ALTER, and other write operations. This is almost always caused by a loss of coordination with ZooKeeper or ClickHouse Keeper.

## Diagnosing the Root Cause

Check the replication status of affected tables:

```sql
SELECT
    database,
    table,
    is_readonly,
    is_session_expired,
    future_parts,
    parts_to_check,
    queue_size,
    last_exception
FROM system.replicas
WHERE is_readonly = 1;
```

The `last_exception` column usually reveals the specific cause.

## Cause 1: ZooKeeper/Keeper Connection Lost

If `last_exception` mentions ZooKeeper or session expired:

```bash
# Check ZooKeeper connectivity
echo ruok | nc zookeeper-host 2181
echo stat | nc zookeeper-host 2181

# Check ClickHouse Keeper
clickhouse-client --query "SELECT * FROM system.zookeeper WHERE path = '/'"
```

Restart the ZooKeeper/Keeper connection:

```sql
SYSTEM RESTART REPLICA database.table_name;
```

Or restart replicas on all tables:

```sql
SYSTEM RESTART REPLICAS;
```

## Cause 2: Disk Space Exhausted

ClickHouse sets tables to readonly when disk space is low. Check disk usage:

```bash
df -h /var/lib/clickhouse
```

Check ClickHouse's view of disk space:

```sql
SELECT
    name,
    path,
    formatReadableSize(free_space) AS free,
    formatReadableSize(total_space) AS total
FROM system.disks;
```

Free up space by removing old parts or old logs:

```sql
-- Force merge to reduce part count
OPTIMIZE TABLE database.table_name FINAL;

-- Check log disk usage
SELECT sum(bytes_on_disk) FROM system.parts WHERE table = 'your_table' AND active = 0;

-- Remove inactive parts
ALTER TABLE database.table_name CLEANUP;
```

## Cause 3: Too Many ZooKeeper Nodes

ReplicatedMergeTree creates ZooKeeper nodes for each data part. If you have millions of small parts, you may hit ZooKeeper limits. Monitor:

```sql
SELECT
    database,
    table,
    zookeeper_path,
    future_parts + queue_size AS pending_parts
FROM system.replicas
ORDER BY pending_parts DESC
LIMIT 10;
```

Run OPTIMIZE to merge parts:

```sql
OPTIMIZE TABLE database.table_name;
```

## Cause 4: Stale Replica

If one replica is far behind, it may enter readonly mode:

```sql
SELECT
    database,
    table,
    replica_is_active
FROM system.replicas
WHERE is_readonly = 1;
```

Force a replica sync:

```sql
SYSTEM SYNC REPLICA database.table_name;
```

## Cause 5: User Permission Restriction

The `readonly` setting in user profiles also prevents writes:

```sql
SELECT name, value FROM system.settings WHERE name = 'readonly';
```

If `readonly=1` or `readonly=2`, the user cannot write. Check the user's profile in `users.xml` or via SQL:

```sql
SHOW CREATE USER current_user();
```

## Summary

"Table is in readonly mode" is primarily caused by ZooKeeper/Keeper connectivity loss, disk space exhaustion, or too many ZooKeeper nodes from an excess of small parts. Diagnose via `system.replicas.last_exception`, restore ZooKeeper connectivity, free disk space, run `OPTIMIZE TABLE` to merge small parts, and use `SYSTEM RESTART REPLICA` to recover.
