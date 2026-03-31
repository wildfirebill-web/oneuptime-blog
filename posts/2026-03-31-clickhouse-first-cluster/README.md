# How to Set Up Your First ClickHouse Cluster

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cluster, Replication, Sharding, High Availability

Description: Learn how to set up a ClickHouse cluster with replication and sharding using ClickHouse Keeper for coordination, with configuration examples.

---

A ClickHouse cluster distributes data across multiple nodes for horizontal scaling and provides replication for fault tolerance. This guide covers setting up a two-shard, two-replica cluster using ClickHouse Keeper as the coordination service.

## Cluster Architecture

A typical production cluster has:
- Multiple shards, each holding a partition of the total data
- Two replicas per shard for redundancy
- ClickHouse Keeper (or ZooKeeper) for distributed coordination

For a minimal cluster: 2 shards x 2 replicas = 4 nodes total.

## Cluster Configuration in config.xml

Each ClickHouse node needs a `remote_servers` configuration. Add this to `/etc/clickhouse-server/config.d/cluster.xml`:

```xml
<clickhouse>
  <remote_servers>
    <my_cluster>
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
    </my_cluster>
  </remote_servers>
</clickhouse>
```

## ClickHouse Keeper Configuration

On each keeper node, add `/etc/clickhouse-server/config.d/keeper.xml`:

```xml
<clickhouse>
  <keeper_server>
    <tcp_port>9181</tcp_port>
    <server_id>1</server_id>
    <raft_configuration>
      <server>
        <id>1</id><hostname>ch-node-1</hostname><port>9234</port>
      </server>
      <server>
        <id>2</id><hostname>ch-node-2</hostname><port>9234</port>
      </server>
      <server>
        <id>3</id><hostname>ch-node-3</hostname><port>9234</port>
      </server>
    </raft_configuration>
  </keeper_server>
</clickhouse>
```

## Creating Replicated Tables

On each node, create the table using `ReplicatedMergeTree`:

```sql
CREATE TABLE events ON CLUSTER my_cluster (
    event_time DateTime,
    event_type String,
    user_id UInt32
) ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/events',
    '{replica}'
)
ORDER BY (event_type, event_time)
PARTITION BY toYYYYMM(event_time);
```

## Creating a Distributed Table

The distributed table is the query entry point:

```sql
CREATE TABLE events_distributed ON CLUSTER my_cluster
AS events
ENGINE = Distributed(my_cluster, default, events, rand());
```

Insert into `events_distributed` and query from it - ClickHouse routes writes to shards and gathers results automatically.

## Verifying the Cluster

```sql
SELECT * FROM system.clusters WHERE cluster = 'my_cluster';
```

## Summary

Setting up a ClickHouse cluster requires configuring `remote_servers` on each node, deploying ClickHouse Keeper for coordination, and creating `ReplicatedMergeTree` tables with shard and replica macros. A `Distributed` table acts as a transparent proxy that routes queries across all shards automatically.
