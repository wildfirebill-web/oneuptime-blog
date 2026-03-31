# How to Use SYSTEM STOP and START for Maintenance Windows

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Maintenance, SYSTEM STOP, SYSTEM START, Operations

Description: Learn how to use ClickHouse SYSTEM STOP and START commands to pause background operations during maintenance windows without stopping the server.

---

ClickHouse runs many background operations - merges, mutations, replication fetches, and more. During maintenance windows (backups, schema changes, disk migrations), pausing these operations prevents conflicts and reduces IO load.

## SYSTEM STOP Commands Overview

ClickHouse provides granular control over background operations:

```sql
-- Stop background merges and mutations
SYSTEM STOP MERGES;

-- Stop replication (stop fetching parts from other replicas)
SYSTEM STOP FETCHES;

-- Stop replication sends (stop sending parts to other replicas)
SYSTEM STOP SENDS;

-- Stop TTL processing
SYSTEM STOP TTL MERGES;

-- Stop moves between volumes (storage tiering)
SYSTEM STOP MOVES;

-- Stop distributed sends
SYSTEM STOP DISTRIBUTED SENDS;
```

## Stopping Operations for a Specific Table

Scope stops to a specific database or table:

```sql
-- Stop merges on a specific table
SYSTEM STOP MERGES my_database.events;

-- Stop TTL merges on a specific table
SYSTEM STOP TTL MERGES my_database.events;
```

## Pre-Backup Maintenance Window

Before running a large backup, pause writes and merges to get a clean state:

```sql
-- Pause background work
SYSTEM STOP MERGES;
SYSTEM STOP TTL MERGES;

-- Flush in-memory data to disk
SYSTEM SYNC FILE CACHE;
SYSTEM FLUSH LOGS;

-- Run backup
BACKUP DATABASE production TO Disk('backups', 'maintenance_backup_2026-03-31/');

-- Resume background work
SYSTEM START MERGES;
SYSTEM START TTL MERGES;
```

## Pre-Schema-Change Maintenance

Before large ALTER TABLE operations, stop competing background work:

```sql
-- Stop merges to prevent conflicts
SYSTEM STOP MERGES my_database.events;

-- Run your ALTER
ALTER TABLE my_database.events ADD COLUMN new_field String DEFAULT '';

-- Wait for mutation to complete
-- Check progress:
SELECT command, is_done, parts_to_do
FROM system.mutations
WHERE table = 'events' AND is_done = 0;

-- Resume merges when done
SYSTEM START MERGES my_database.events;
```

## Checking What Is Currently Stopped

```sql
-- View current state of background operations
SELECT metric, value
FROM system.metrics
WHERE metric IN (
    'BackgroundMergesAndMutationsPoolTask',
    'BackgroundFetchesPoolTask',
    'BackgroundMovesPoolTask',
    'BackgroundDistributedSendsPoolTask'
);
```

## SYSTEM STOP vs. READ ONLY Mode

SYSTEM STOP commands pause background operations but still accept queries and inserts. For full maintenance mode that blocks writes:

```sql
-- Set table to read-only during maintenance
ALTER TABLE my_database.events MODIFY SETTING parts_to_throw_insert = 0;

-- Restore after maintenance
ALTER TABLE my_database.events MODIFY SETTING parts_to_throw_insert = 300;
```

## Restarting All Background Operations

After maintenance, restart everything at once:

```sql
SYSTEM START MERGES;
SYSTEM START TTL MERGES;
SYSTEM START FETCHES;
SYSTEM START SENDS;
SYSTEM START MOVES;
SYSTEM START DISTRIBUTED SENDS;
```

## Verifying Background Operations Resume

Check that background tasks are running again after restarting:

```sql
-- Check merges are running
SELECT table, elapsed, progress FROM system.merges LIMIT 5;

-- Check replication queue is processing
SELECT database, table, type, create_time FROM system.replication_queue LIMIT 5;
```

## Summary

ClickHouse's SYSTEM STOP and START commands let you pause specific background operations without stopping the server, making them ideal for maintenance windows. Stop merges before large backups or schema changes, flush logs and file cache before snapshots, and always resume operations after maintenance is complete. Scope stops to specific tables when possible to minimize impact on other workloads.
