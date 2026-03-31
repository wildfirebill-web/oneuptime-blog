# How to Set Up Active-Active ClickHouse Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cluster, High Availability, Replication, Distributed

Description: Learn how to configure an active-active ClickHouse cluster where multiple nodes serve both reads and writes simultaneously for high availability.

---

## What Is an Active-Active Cluster?

In an active-active setup, all nodes accept both reads and writes at the same time. This contrasts with active-passive, where standby nodes only take over during failure. Active-active provides higher throughput and no idle capacity.

## Architecture Overview

A typical active-active ClickHouse cluster uses:

- 2 or more shards, each with 2 replicas
- ZooKeeper or ClickHouse Keeper for coordination
- Distributed tables that route queries to all shards

```text
Client
  |
  +-- Distributed Table
        |          |
      Shard 1    Shard 2
      /    \     /    \
  Rep1  Rep2  Rep3  Rep4
```

## Cluster Configuration in config.xml

```xml
<remote_servers>
  <production_cluster>
    <shard>
      <replica>
        <host>ch-node-1</host>
        <port>9000</port>
      </replica>
      <replica>
        <host>ch-node-2</host>
        <port>9000</port>
      </replica>
    </shard>
    <shard>
      <replica>
        <host>ch-node-3</host>
        <port>9000</port>
      </replica>
      <replica>
        <host>ch-node-4</host>
        <port>9000</port>
      </replica>
    </shard>
  </production_cluster>
</remote_servers>
```

## Creating Replicated Tables on Each Node

Run this on each replica, changing the ZooKeeper path and replica name:

```sql
CREATE TABLE events_local
(
    id         UInt64,
    event_type String,
    created_at DateTime
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/events',
    '{replica}'
)
PARTITION BY toYYYYMM(created_at)
ORDER BY (event_type, created_at);
```

## Creating the Distributed Table

```sql
CREATE TABLE events_distributed
ON CLUSTER production_cluster
AS events_local
ENGINE = Distributed(
    'production_cluster',
    currentDatabase(),
    'events_local',
    sipHash64(id)
);
```

## Writing to the Cluster

Insert into the Distributed table to fan out across shards:

```sql
INSERT INTO events_distributed SELECT * FROM incoming_batch;
```

Each node writes to its local replica and replicates to the other replica in the same shard.

## Reading from the Cluster

```sql
SELECT event_type, count()
FROM events_distributed
GROUP BY event_type;
```

ClickHouse automatically queries all shards and merges results.

## Monitoring Replication Health

```sql
SELECT
    database,
    table,
    is_leader,
    total_replicas,
    active_replicas
FROM system.replicas
WHERE is_readonly = 0;
```

## Summary

An active-active ClickHouse cluster uses `ReplicatedMergeTree` on each node and a `Distributed` table as the query entry point. All nodes serve reads and writes, providing horizontal scaling and fault tolerance. Monitor `system.replicas` regularly to catch any replication lag before it affects query freshness.
