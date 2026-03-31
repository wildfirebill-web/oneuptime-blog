# How to Configure Distributed DDL in ClickHouse Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Distributed DDL, ON CLUSTER, ZooKeeper, Administration

Description: Learn how to configure and use distributed DDL in ClickHouse to run schema changes across all cluster nodes with a single ON CLUSTER query.

---

Distributed DDL in ClickHouse lets you run `CREATE`, `DROP`, `ALTER`, and other schema operations across all nodes in a cluster with a single query. This is coordinated through ZooKeeper or ClickHouse Keeper, which tracks the DDL task queue.

## How Distributed DDL Works

When you run a DDL statement with `ON CLUSTER`, ClickHouse stores the task in ZooKeeper at the path `/clickhouse/task_queue/ddl/`. Each node in the cluster watches this path and executes the task locally when it appears.

## Enabling Distributed DDL

Distributed DDL is enabled by default if a ZooKeeper or Keeper connection is configured. Verify your ZooKeeper config in `config.xml`:

```xml
<zookeeper>
    <node>
        <host>zk-node-01</host>
        <port>2181</port>
    </node>
    <node>
        <host>zk-node-02</host>
        <port>2181</port>
    </node>
    <node>
        <host>zk-node-03</host>
        <port>2181</port>
    </node>
</zookeeper>
```

## Configuring the DDL Task Path

By default, ClickHouse uses `/clickhouse/task_queue/ddl` in ZooKeeper. Customize it in `config.xml`:

```xml
<distributed_ddl>
    <path>/clickhouse/task_queue/ddl</path>
    <pool_size>1</pool_size>
    <task_max_lifetime>604800</task_max_lifetime>
    <cleanup_delay_period>60</cleanup_delay_period>
    <max_tasks_in_queue>1000</max_tasks_in_queue>
</distributed_ddl>
```

Key settings:
- `task_max_lifetime` - seconds to keep completed tasks in ZooKeeper (default 7 days)
- `max_tasks_in_queue` - maximum DDL tasks to accumulate before rejecting new ones
- `pool_size` - number of DDL worker threads per node

## Running a Distributed DDL Statement

```sql
-- Create a table on all nodes in the cluster
CREATE TABLE events_local ON CLUSTER my_cluster
(
    event_time DateTime,
    user_id UInt64,
    action LowCardinality(String)
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events_local', '{replica}')
PARTITION BY toYYYYMM(event_time)
ORDER BY (user_id, event_time);
```

## Setting DDL Timeout

Control how long to wait for all nodes to complete a distributed DDL:

```sql
SET distributed_ddl_task_timeout = 300;

ALTER TABLE events_local ON CLUSTER my_cluster
ADD COLUMN page_url String DEFAULT '';
```

If a node is down and does not execute the DDL within the timeout, the query returns an error for that node but the task remains in ZooKeeper. The node will execute it when it comes back online.

## Monitoring DDL Task Queue

Check pending and completed DDL tasks:

```sql
SELECT *
FROM system.zookeeper
WHERE path = '/clickhouse/task_queue/ddl'
ORDER BY name DESC
LIMIT 10;
```

View task details including which nodes completed execution:

```sql
SELECT *
FROM system.zookeeper
WHERE path = '/clickhouse/task_queue/ddl/query-0000001234'
ORDER BY name;
```

## Handling Failed DDL Tasks

If a node misses a DDL task due to downtime, it will execute the task on restart. You can also manually trigger DDL replication:

```sql
-- Check DDL log for missed tasks
SELECT *
FROM system.ddl_worker_log
WHERE status != 'Finished'
ORDER BY entry_time DESC;
```

## Summary

Distributed DDL in ClickHouse coordinates schema changes across all cluster nodes via ZooKeeper. Configure the task queue path, lifetime, and timeout to match your cluster's scale. Use `ON CLUSTER` for all schema operations, set a generous `distributed_ddl_task_timeout` for large clusters, and monitor `system.ddl_worker_log` for nodes that missed executions.
