# How ClickHouse Replication Protocol Works

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Replication, ZooKeeper, ClickHouse Keeper, High Availability

Description: Understand how ClickHouse replicates data between replicas using ZooKeeper or ClickHouse Keeper as a coordination layer, and how parts are transferred between nodes.

---

## Replication Architecture

ClickHouse replication uses `ReplicatedMergeTree` engines and a coordination service (ZooKeeper or ClickHouse Keeper) to keep replicas in sync. The coordination service stores metadata - it does not store actual data. Data is transferred directly between ClickHouse nodes.

## Replication Log in ZooKeeper

Every operation that changes data (INSERT, MERGE, MUTATION) is recorded as a log entry in ZooKeeper under the table's ZooKeeper path:

```text
/clickhouse/tables/{shard}/{table}/log/log-0000000042
```

Each replica watches this log and pulls entries to execute. This means replicas can lag behind the leader but will eventually converge.

## Insert Flow

1. Client sends INSERT to any replica
2. That replica writes the part locally and creates a log entry in ZooKeeper
3. Other replicas see the new log entry and fetch the part either:
   - From the replica that wrote it (direct part fetch over HTTP)
   - By replaying the INSERT themselves if they have the data

```sql
-- Check replication lag
SELECT database, table, replica_name, absolute_delay
FROM system.replicas
WHERE absolute_delay > 10;
```

## Part Fetching

When a replica needs a part from another replica, it uses ClickHouse's part fetch protocol over HTTP:

```text
GET /?action=sendPart&part=20240101_1_5_2&database=default&table=events
```

The response streams the compressed part files directly. The receiving replica writes them to a temporary directory and atomically renames on success.

## Merge Coordination

To avoid duplicating merge work, only one replica executes a given merge. The replica assigns itself a merge log entry in ZooKeeper:

```text
/clickhouse/tables/{shard}/{table}/log/log-0000000043 (type: MERGE_PARTS)
```

Other replicas see this log entry and either wait for the merged part to be fetched, or execute the merge themselves if the source replica is slow.

## Quorum Inserts

For strong durability, use quorum inserts so ClickHouse confirms success only after N replicas have written the part:

```sql
SET insert_quorum = 2;
SET insert_quorum_timeout = 60000;

INSERT INTO events VALUES (...);
```

## Monitoring Replication

```sql
-- Full replication status
SELECT
    replica_name,
    is_leader,
    queue_size,
    absolute_delay,
    total_replicas,
    active_replicas
FROM system.replicas
WHERE table = 'events';
```

## ClickHouse Keeper

ClickHouse Keeper is a built-in alternative to ZooKeeper, written in C++ and integrated into ClickHouse. It uses the Raft consensus algorithm and is recommended for new deployments:

```xml
<keeper_server>
    <tcp_port>9181</tcp_port>
    <server_id>1</server_id>
    <raft_configuration>
        <server><id>1</id><hostname>ch1</hostname><port>9444</port></server>
        <server><id>2</id><hostname>ch2</hostname><port>9444</port></server>
        <server><id>3</id><hostname>ch3</hostname><port>9444</port></server>
    </raft_configuration>
</keeper_server>
```

## Summary

ClickHouse replication works by logging every data-changing operation to ZooKeeper or Keeper, then having each replica pull and execute those operations. Part data transfers happen directly between nodes over HTTP. Quorum inserts provide stronger durability guarantees at the cost of latency.
