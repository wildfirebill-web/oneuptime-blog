# How to Configure ClickHouse Remote Servers for Clusters

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Clustering, Remote Servers, Distributed, Configuration

Description: Learn how to configure ClickHouse remote_servers to define cluster topology for Distributed tables and horizontal scaling across multiple shards and replicas.

---

## Overview

ClickHouse uses the `<remote_servers>` configuration section to define cluster topology - which servers form each shard and which servers are replicas within each shard. This configuration enables `Distributed` tables to route queries and inserts across the cluster.

## Basic Cluster Configuration

Define a cluster named `my_cluster` with 2 shards and 2 replicas each:

```xml
<!-- /etc/clickhouse-server/config.d/clusters.xml -->
<clickhouse>
    <remote_servers>
        <my_cluster>
            <!-- Shard 1 -->
            <shard>
                <weight>1</weight>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>ch-node-1</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>ch-node-2</host>
                    <port>9000</port>
                </replica>
            </shard>
            <!-- Shard 2 -->
            <shard>
                <weight>1</weight>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>ch-node-3</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>ch-node-4</host>
                    <port>9000</port>
                </replica>
            </shard>
        </my_cluster>
    </remote_servers>
</clickhouse>
```

Deploy this configuration to all nodes in the cluster.

## Single-Shard Cluster for Replication Only

For a replicated setup without sharding:

```xml
<remote_servers>
    <replicated_cluster>
        <shard>
            <internal_replication>true</internal_replication>
            <replica>
                <host>ch-primary</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>ch-replica-1</host>
                <port>9000</port>
            </replica>
        </shard>
    </replicated_cluster>
</remote_servers>
```

## Shard Weight

`<weight>` controls the proportion of data routed to each shard during distributed inserts. Useful for asymmetric hardware:

```xml
<shard>
    <weight>2</weight>  <!-- Gets 2x as many inserts as weight=1 shards -->
    ...
</shard>
```

## internal_replication

When `<internal_replication>true</internal_replication>`, distributed inserts write to only one replica per shard and rely on ClickHouse's built-in replication (via ReplicatedMergeTree and ZooKeeper/ClickHouse Keeper) to propagate to other replicas.

When `false`, the Distributed engine writes to all replicas directly - useful if the underlying tables are not replicated.

## Authentication for Remote Connections

Add credentials for inter-node connections:

```xml
<replica>
    <host>ch-node-1</host>
    <port>9000</port>
    <user>cluster_user</user>
    <password>secret</password>
</replica>
```

## Creating a Distributed Table

After defining the cluster, create a Distributed table on each node:

```sql
-- First create the local table on each node
CREATE TABLE events_local ON CLUSTER my_cluster
(
    event_id   UInt64,
    event_time DateTime,
    user_id    UInt64,
    action     String
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
ORDER BY (user_id, event_time)
PARTITION BY toYYYYMM(event_time);

-- Then create the distributed table
CREATE TABLE events ON CLUSTER my_cluster
(
    event_id   UInt64,
    event_time DateTime,
    user_id    UInt64,
    action     String
)
ENGINE = Distributed(my_cluster, default, events_local, rand());
```

## Verifying Cluster Configuration

```sql
SELECT * FROM system.clusters WHERE cluster = 'my_cluster';
```

```sql
SELECT
    cluster,
    shard_num,
    replica_num,
    host_name,
    port,
    is_local
FROM system.clusters
WHERE cluster = 'my_cluster'
ORDER BY shard_num, replica_num
```

## Testing Connectivity

```sql
SELECT * FROM remote('ch-node-2:9000', system.one) SETTINGS receive_timeout = 5
```

## Macros for Multi-Node Config

Use `<macros>` to define node-specific shard/replica values referenced in table DDL:

```xml
<!-- On ch-node-1 (shard 1, replica 1) -->
<clickhouse>
    <macros>
        <shard>01</shard>
        <replica>ch-node-1</replica>
    </macros>
</clickhouse>
```

## Summary

The `<remote_servers>` section defines ClickHouse cluster topology with named clusters, shards, and replicas. Configure `internal_replication = true` when using ReplicatedMergeTree, set shard weights for asymmetric hardware, and reference cluster names in `Distributed` table engines. Verify topology with `system.clusters` and use macros for clean multi-node DDL.
