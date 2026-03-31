# How to Fix 'Replica is not in sync' Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Replication, Replica, Error, Troubleshooting

Description: Diagnose and fix 'Replica is not in sync' errors in ClickHouse by monitoring replication lag, syncing replicas, and resolving diverged data.

---

"Replica is not in sync" errors in ClickHouse indicate that a replica has fallen behind the leader and is missing parts or mutations. This can cause inconsistent query results across replicas and blocks certain DDL operations.

## Check Replication Status

```sql
SELECT
    database,
    table,
    is_leader,
    is_readonly,
    absolute_delay,
    queue_size,
    inserts_in_queue,
    merges_in_queue,
    last_queue_update_exception
FROM system.replicas
ORDER BY absolute_delay DESC;
```

High `absolute_delay` or `queue_size` indicates replication lag.

## View the Replication Queue

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
ORDER BY create_time;
```

Entries with repeated failures show the root cause.

## Sync a Lagging Replica

Force the replica to sync:

```sql
SYSTEM SYNC REPLICA my_database.my_table;
```

Wait for the queue to drain:

```sql
SYSTEM SYNC REPLICA my_database.my_table STRICT;
-- Waits until queue is empty or throws error
```

## Resolve a Stuck Replication Entry

If a single entry is stuck due to a missing part:

```sql
-- Clear the entry causing the block
SYSTEM DROP REPLICA 'replica_name' FROM TABLE my_database.my_table;
```

Then let the replica re-fetch from the leader:

```sql
SYSTEM RESTART REPLICA my_database.my_table;
```

## Fetch Missing Parts from Another Replica

```sql
ALTER TABLE my_database.my_table FETCH PARTITION 'partition_id'
FROM 'zookeeper_path_to_other_replica';
```

## Full Resync of a Badly Diverged Replica

For a replica severely out of sync, a full resync is safest:

```sql
-- On the lagging replica:
DETACH TABLE my_database.my_table;
```

Remove local data:

```bash
sudo rm -rf /var/lib/clickhouse/data/my_database/my_table/
```

```sql
ATTACH TABLE my_database.my_table;
```

ClickHouse will download all parts from another replica.

## Summary

"Replica is not in sync" is diagnosed via `system.replicas` and `system.replication_queue`. Use `SYSTEM SYNC REPLICA` to force synchronization, clear stuck entries with `SYSTEM DROP REPLICA`, and for severely diverged replicas, detach, remove local data, and reattach to trigger a clean re-download from healthy replicas.
