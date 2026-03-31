# How to Set Up ClickHouse Cluster with Docker Compose

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Docker, Docker Compose, Cluster, Replication, ClickHouse Keeper

Description: Learn how to set up a multi-node ClickHouse cluster with replication using Docker Compose, including ClickHouse Keeper for coordination.

---

Running a ClickHouse cluster in Docker Compose is useful for testing distributed query behavior, replication, and sharding before deploying to production. This guide sets up a 2-shard, 2-replica cluster with ClickHouse Keeper.

## Architecture

```text
clickhouse-01 (shard 1, replica 1)
clickhouse-02 (shard 1, replica 2)
clickhouse-03 (shard 2, replica 1)
clickhouse-04 (shard 2, replica 2)
keeper-01, keeper-02, keeper-03 (ClickHouse Keeper quorum)
```

## docker-compose.yml

```yaml
version: '3.8'

services:
  keeper-01:
    image: clickhouse/clickhouse-keeper:latest
    container_name: keeper-01
    volumes:
      - ./config/keeper-01.xml:/etc/clickhouse-keeper/keeper_config.xml
    ports:
      - "9181:9181"

  clickhouse-01:
    image: clickhouse/clickhouse-server:latest
    container_name: clickhouse-01
    depends_on: [keeper-01]
    volumes:
      - ./config/clickhouse-01.xml:/etc/clickhouse-server/config.d/cluster.xml
      - ch01_data:/var/lib/clickhouse
    ports:
      - "8123:8123"
      - "9000:9000"

  clickhouse-02:
    image: clickhouse/clickhouse-server:latest
    container_name: clickhouse-02
    depends_on: [keeper-01]
    volumes:
      - ./config/clickhouse-02.xml:/etc/clickhouse-server/config.d/cluster.xml
      - ch02_data:/var/lib/clickhouse

volumes:
  ch01_data:
  ch02_data:
```

## Cluster Configuration (config/clickhouse-01.xml)

```xml
<clickhouse>
  <remote_servers>
    <my_cluster>
      <shard>
        <replica>
          <host>clickhouse-01</host>
          <port>9000</port>
        </replica>
        <replica>
          <host>clickhouse-02</host>
          <port>9000</port>
        </replica>
      </shard>
    </my_cluster>
  </remote_servers>

  <zookeeper>
    <node>
      <host>keeper-01</host>
      <port>9181</port>
    </node>
  </zookeeper>

  <macros>
    <shard>01</shard>
    <replica>clickhouse-01</replica>
  </macros>
</clickhouse>
```

## Creating a Replicated Table

```sql
CREATE TABLE events ON CLUSTER my_cluster (
    event_id UInt64,
    event_type String,
    event_date Date
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, event_id);
```

## Verifying Replication

```sql
SELECT
    database,
    table,
    replica_name,
    is_leader,
    total_replicas,
    active_replicas
FROM system.replicas
WHERE table = 'events';
```

## Summary

A Docker Compose ClickHouse cluster requires separate configuration files for each node defining the cluster topology, macros, and Keeper connection. Use `ReplicatedMergeTree` tables with `ON CLUSTER` DDL to create replicated tables across all nodes simultaneously.
