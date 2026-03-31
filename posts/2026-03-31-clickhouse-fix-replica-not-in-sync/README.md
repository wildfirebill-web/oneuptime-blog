# How to Fix 'Replica is not in sync' Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Troubleshooting, Replication, Replica, Synchronization

Description: Fix 'Replica is not in sync' errors in ClickHouse by diagnosing replication lag, forcing synchronization, and recovering stale or detached replicas.

---

The "Replica is not in sync" error indicates a ReplicatedMergeTree replica has fallen behind the leader and cannot serve consistent reads or writes. This can be caused by network issues, hardware failures, ZooKeeper downtime, or overwhelming replication queue.

## Diagnosing the Sync State

Check replication status across all tables:

```sql
SELECT
    database,
    table,
    is_leader,
    total_replicas,
    active_replicas,
    queue_size,
    queue_oldest_time,
    absolute_delay,
    last_exception
FROM system.replicas
WHERE absolute_delay > 0 OR is_readonly = 1
ORDER BY absolute_delay DESC;
```

Key columns:
- `absolute_delay` - seconds behind the leader
- `queue_size` - number of operations waiting to be applied
- `last_exception` - why the replica is stuck

## Fix 1: Force Replica Sync

Force a replica to catch up:

```sql
SYSTEM SYNC REPLICA mydb.my_table;
```

Wait for it to complete (can take time if the queue is large):

```sql
-- Monitor progress
SELECT queue_size, absolute_delay
FROM system.replicas
WHERE database = 'mydb' AND table = 'my_table';
```

## Fix 2: Restart the Replica

If the replica is stuck due to a transient error:

```sql
SYSTEM RESTART REPLICA mydb.my_table;
```

This reconnects to ZooKeeper/Keeper and resets the replication queue processing.

## Fix 3: Check and Fix the Replication Queue

View what is in the queue:

```sql
SELECT
    database,
    table,
    type,
    create_time,
    required_quorum,
    source_replica,
    last_exception,
    num_tries
FROM system.replication_queue
WHERE last_exception != ''
ORDER BY create_time
LIMIT 20;
```

Remove a stuck queue entry if it is safe to do so:

```sql
-- Use with caution - only if you understand what is being dropped
ALTER TABLE mydb.my_table DROP DETACHED PART 'part_name';
```

## Fix 4: Recover a Stale Replica

If a replica is very far behind and cannot recover normally, re-initialize it:

```bash
# Stop ClickHouse on the stale replica
sudo systemctl stop clickhouse-server

# Remove the replica's data (it will re-sync from leader)
rm -rf /var/lib/clickhouse/data/mydb/my_table/

# Start ClickHouse - it will download missing parts from the leader
sudo systemctl start clickhouse-server
```

Then in ClickHouse:

```sql
SYSTEM SYNC REPLICA mydb.my_table;
```

## Fix 5: Check for Excessive Part Count

Too many parts can slow replication to a crawl:

```sql
SELECT
    database,
    table,
    count() AS parts
FROM system.parts
WHERE active = 1
GROUP BY database, table
HAVING count() > 300
ORDER BY count() DESC;
```

Force a merge to reduce part count:

```sql
OPTIMIZE TABLE mydb.my_table;
```

## Monitoring Replication Health

Set up regular monitoring:

```sql
SELECT
    database,
    table,
    absolute_delay,
    queue_size
FROM system.replicas
WHERE absolute_delay > 60
   OR queue_size > 100;
```

Alert when `absolute_delay > 300` (5 minutes behind).

## Summary

"Replica is not in sync" errors are fixed by using `SYSTEM SYNC REPLICA` for minor lag, `SYSTEM RESTART REPLICA` for stuck queue processing, and manual re-initialization for severely stale replicas. Prevent recurrence by monitoring `system.replicas.absolute_delay`, running periodic `OPTIMIZE TABLE` to control part count, and ensuring ZooKeeper/Keeper connectivity is stable.
