# How to Configure ClickHouse Distributed DDL Settings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Distributed DDL, Clustering, ON CLUSTER, Configuration

Description: Learn how to configure ClickHouse distributed DDL settings that control how ON CLUSTER DDL statements are executed and synchronized across cluster nodes.

---

## Overview

ClickHouse's distributed DDL allows you to run `CREATE`, `ALTER`, `DROP`, and other DDL statements across all cluster nodes using `ON CLUSTER`. These operations are coordinated via ZooKeeper or ClickHouse Keeper. The `<distributed_ddl>` configuration section controls the behavior, timeouts, and storage path for these distributed tasks.

## Basic Distributed DDL Configuration

```xml
<!-- /etc/clickhouse-server/config.d/distributed-ddl.xml -->
<clickhouse>
    <distributed_ddl>
        <!-- ZooKeeper/Keeper path for DDL task queue -->
        <path>/clickhouse/task_queue/ddl</path>

        <!-- How long to wait for all cluster nodes to complete DDL (seconds) -->
        <task_max_lifetime>604800</task_max_lifetime>  <!-- 7 days -->

        <!-- Maximum number of DDL tasks to keep in the queue -->
        <max_tasks_in_queue>1000</max_tasks_in_queue>

        <!-- How often to clean up completed tasks (seconds) -->
        <cleanup_delay_period>60</cleanup_delay_period>
    </distributed_ddl>
</clickhouse>
```

## Running ON CLUSTER DDL

With distributed DDL configured, use `ON CLUSTER` to execute DDL across all nodes:

```sql
-- Create a table on all nodes in my_cluster
CREATE TABLE events_local ON CLUSTER my_cluster
(
    event_id   UInt64,
    event_time DateTime,
    user_id    UInt64
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
ORDER BY (user_id, event_time);
```

```sql
-- Add a column on all nodes
ALTER TABLE events_local ON CLUSTER my_cluster
    ADD COLUMN session_id String DEFAULT '';
```

```sql
-- Drop a table on all nodes
DROP TABLE IF EXISTS events_local ON CLUSTER my_cluster;
```

## Monitoring DDL Task Status

View pending and completed distributed DDL tasks:

```sql
SELECT
    entry,
    status,
    cluster,
    query,
    initiator_host,
    create_time,
    finish_time
FROM system.distributed_ddl_queue
ORDER BY create_time DESC
LIMIT 20
```

Check for failed tasks:

```sql
SELECT *
FROM system.distributed_ddl_queue
WHERE status = 'Active'
  AND create_time < now() - INTERVAL 1 HOUR
```

## DDL Wait Timeout

Control how long the client waits for all nodes to complete:

```sql
-- Set a timeout for distributed DDL completion
SET distributed_ddl_task_timeout = 180;  -- 180 seconds

CREATE TABLE my_table ON CLUSTER my_cluster (...) ENGINE = ...;
```

If some nodes don't respond within the timeout, the client returns a warning but the task continues executing on remaining nodes.

## Handling DDL on Unavailable Nodes

When a node is temporarily down, the DDL task remains in the queue. When the node comes back online, it reads the pending tasks from ZooKeeper and executes them automatically.

Check what tasks a node is catching up on:

```sql
SELECT entry, query, status, create_time
FROM system.distributed_ddl_queue
WHERE status IN ('Active', 'Failed')
ORDER BY create_time
```

## Retry Failed Tasks

To retry a failed DDL task, you typically need to re-execute the original DDL statement. For idempotent operations like `CREATE TABLE IF NOT EXISTS` or `ALTER TABLE ADD COLUMN IF NOT EXISTS`, this is safe:

```sql
ALTER TABLE events_local ON CLUSTER my_cluster
    ADD COLUMN IF NOT EXISTS new_col String DEFAULT '';
```

## Disabling Distributed DDL for a Session

If you want to run DDL on the local node only without cluster-wide propagation:

```sql
SET allow_distributed_ddl = 0;
CREATE TABLE local_only_table (...) ENGINE = MergeTree() ORDER BY id;
```

## ZooKeeper Path Structure

The DDL task queue path should be unique per cluster:

```xml
<distributed_ddl>
    <path>/clickhouse/task_queue/ddl</path>
</distributed_ddl>
```

If you run multiple ClickHouse clusters sharing the same ZooKeeper ensemble, use distinct paths:

```xml
<path>/clickhouse/cluster_a/task_queue/ddl</path>
```

## Summary

ClickHouse distributed DDL coordinates schema changes across cluster nodes via ZooKeeper using a task queue stored at the configured path. Set appropriate timeouts with `distributed_ddl_task_timeout`, monitor pending tasks with `system.distributed_ddl_queue`, and use `IF NOT EXISTS`/`IF EXISTS` clauses for idempotent DDL that can be safely re-executed when nodes catch up after downtime.
