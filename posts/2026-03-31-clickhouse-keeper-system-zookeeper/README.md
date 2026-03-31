# How to Monitor ClickHouse Keeper with system.zookeeper

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ClickHouse Keeper, Monitoring, ZooKeeper, Database

Description: Learn how to use the system.zookeeper table to inspect the ClickHouse Keeper data tree, verify replication metadata, and diagnose coordination issues.

`system.zookeeper` is a virtual table in ClickHouse that lets you browse and query the Keeper (or ZooKeeper) data tree using SQL. It is your window into the coordination layer, showing you exactly what metadata ClickHouse is storing, which replicas are active, and what replication state looks like from Keeper's perspective. This is invaluable for debugging replication issues.

## Basic Usage

The `system.zookeeper` table requires a `WHERE path = '...'` filter. Without it, ClickHouse does not know which ZooKeeper node to list:

```sql
-- List the top-level nodes under the ClickHouse root
SELECT
    name,
    value,
    czxid,
    mzxid,
    ctime,
    mtime,
    dataLength,
    numChildren,
    ephemeralOwner,
    pzxid
FROM system.zookeeper
WHERE path = '/clickhouse'
ORDER BY name;
```

## Exploring the Replication Tree

The standard ZooKeeper path structure for ClickHouse replication is:

```text
/clickhouse
  /tables
    /{shard}
      /{table_name}
        /replicas
          /replica1
          /replica2
        /log
        /blocks
        /parts
        /leader_election
        /mutations
```

Navigate this tree to understand what ClickHouse stores:

```sql
-- List all shards
SELECT name
FROM system.zookeeper
WHERE path = '/clickhouse/tables'
ORDER BY name;

-- List all tables in shard 01
SELECT name
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01'
ORDER BY name;

-- List all replicas for a table
SELECT
    name AS replica,
    value,
    ctime AS registered_at,
    mtime AS last_modified
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/events/replicas'
ORDER BY name;
```

## Checking Which Replica is the Leader

```sql
-- Check leader election for a table
SELECT
    name,
    value,
    ephemeralOwner
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/events/leader_election'
ORDER BY name;
```

The replica holding the leader election ephemeral node is the current leader. If no nodes appear, there is no leader - which means replication is not functioning.

## Inspecting Replica State

Each replica has its own subtree under `/replicas/{replica_name}`:

```sql
-- Check the state of a specific replica
SELECT
    name,
    value,
    dataLength
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/events/replicas/ch1.internal'
ORDER BY name;

-- The 'is_active' node tells you if the replica is currently alive
SELECT
    name,
    value,
    ephemeralOwner
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/events/replicas/ch1.internal'
  AND name = 'is_active';
```

An `is_active` node with `ephemeralOwner != 0` means the replica is connected and its ZooKeeper session is alive. If the node does not exist or `ephemeralOwner = 0`, the replica is offline.

## Viewing the Replication Log

The replication log is stored in ZooKeeper and contains every operation that needs to be applied to all replicas:

```sql
-- Count entries in the replication log
SELECT count() AS log_entries
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/events/log';

-- List recent log entries
SELECT
    name,
    value,
    ctime AS created
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/events/log'
ORDER BY name DESC
LIMIT 20;
```

The log grows as new parts are added and merges happen. Each replica has a pointer (`log_pointer`) to its position in this log. The difference between the log's maximum index and a replica's `log_pointer` tells you how far behind the replica is.

## Checking Block Deduplication

ClickHouse stores block checksums in ZooKeeper for insert deduplication:

```sql
-- Count deduplication blocks stored (last N inserts)
SELECT count() AS dedup_blocks
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/events/blocks';

-- Check a specific block
SELECT
    name,
    value,
    ctime AS inserted_at
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/events/blocks'
ORDER BY ctime DESC
LIMIT 10;
```

If the blocks node has too many entries, it may indicate the `replicated_deduplication_window` setting is too high.

## Diagnosing a Stuck Replication Queue

When replication seems stuck, inspect the queue directly in ZooKeeper:

```sql
-- Check how many queue entries a replica has
SELECT
    count() AS queue_length
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/events/replicas/ch1.internal/queue';

-- Inspect queue entries
SELECT
    name,
    value,
    ctime AS queued_at
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/events/replicas/ch1.internal/queue'
ORDER BY name ASC
LIMIT 20;
```

Compare this with `system.replication_queue` to see if the local view matches ZooKeeper.

## Monitoring Mutations

```sql
-- List pending mutations in ZooKeeper
SELECT
    name,
    value,
    ctime AS mutation_started
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/events/mutations'
ORDER BY name;
```

## Checking the DDL Task Queue

The distributed DDL task queue is also stored in ZooKeeper:

```sql
-- List pending DDL tasks
SELECT
    name,
    value,
    ctime AS task_created
FROM system.zookeeper
WHERE path = '/clickhouse/task_queue/ddl'
ORDER BY name DESC
LIMIT 10;

-- Inspect a specific task
SELECT name, value
FROM system.zookeeper
WHERE path = '/clickhouse/task_queue/ddl/query-0000000001'
ORDER BY name;
```

## Full Health Check Query

A single query that checks the health of all replicas across all tables:

```sql
-- Check if all replicas are active by comparing ZooKeeper to system.replicas
SELECT
    r.database,
    r.table,
    r.replica_name,
    r.is_readonly,
    r.absolute_delay,
    r.active_replicas,
    r.total_replicas,
    CASE
        WHEN r.absolute_delay > 60  THEN 'LAGGING'
        WHEN r.is_readonly = 1      THEN 'READ_ONLY'
        WHEN r.active_replicas < r.total_replicas THEN 'DEGRADED'
        ELSE 'OK'
    END AS status
FROM system.replicas AS r
ORDER BY r.absolute_delay DESC;
```

## Checking the ZooKeeper Connection

```sql
-- Check the ClickHouse-to-Keeper connection
SELECT *
FROM system.zookeeper_connection;

-- Sample output:
-- name: zookeeper
-- host: keeper1.internal
-- port: 2181
-- index: 0
-- connected_time: 2026-03-31 10:00:00
-- session_uptime_elapsed_seconds: 3600
-- is_expired: 0
-- keeper_api_version: 3
-- client_id: 123456789
```

If `is_expired = 1`, the session has expired and ClickHouse is trying to reconnect. All replicated tables on this node will be in read-only mode until the session is re-established.
