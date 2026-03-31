# How to Set Up Three-Node ClickHouse Replication

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Replication, ReplicatedMergeTree, ClickHouse Keeper, Cluster, High Availability

Description: Configure three-node ClickHouse replication with a built-in Keeper ensemble for a production-grade setup with no single point of failure.

---

A three-node ClickHouse replication setup is the minimum recommended configuration for production. Three nodes allow the Keeper ensemble to maintain quorum even if one node fails, and all three can serve read queries.

## Architecture

- ch1, ch2, ch3: all three are replicas of shard 1
- ClickHouse Keeper runs on all three nodes (forms a three-node Raft ensemble)
- No external ZooKeeper needed

## Configure Keeper on All Three Nodes

Add `/etc/clickhouse-server/config.d/keeper.xml` on each node:

```xml
<clickhouse>
  <keeper_server>
    <tcp_port>9181</tcp_port>
    <server_id>1</server_id>  <!-- 1, 2, or 3 per node -->
    <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
    <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>
    <coordination_settings>
      <operation_timeout_ms>10000</operation_timeout_ms>
      <session_timeout_ms>30000</session_timeout_ms>
      <raft_logs_level>warning</raft_logs_level>
    </coordination_settings>
    <raft_configuration>
      <server>
        <id>1</id><hostname>ch1</hostname><port>9234</port>
      </server>
      <server>
        <id>2</id><hostname>ch2</hostname><port>9234</port>
      </server>
      <server>
        <id>3</id><hostname>ch3</hostname><port>9234</port>
      </server>
    </raft_configuration>
  </keeper_server>
</clickhouse>
```

## Configure Cluster Topology

Add `/etc/clickhouse-server/config.d/cluster.xml` on all nodes:

```xml
<clickhouse>
  <remote_servers>
    <production>
      <shard>
        <replica><host>ch1</host><port>9000</port></replica>
        <replica><host>ch2</host><port>9000</port></replica>
        <replica><host>ch3</host><port>9000</port></replica>
      </shard>
    </production>
  </remote_servers>

  <zookeeper>
    <node><host>ch1</host><port>9181</port></node>
    <node><host>ch2</host><port>9181</port></node>
    <node><host>ch3</host><port>9181</port></node>
  </zookeeper>

  <macros>
    <shard>01</shard>
    <replica>ch1</replica>  <!-- unique per node -->
  </macros>
</clickhouse>
```

## Create Replicated Tables

```sql
CREATE TABLE metrics ON CLUSTER production (
    ts         DateTime,
    host       String,
    metric     String,
    value      Float64
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/metrics', '{replica}')
PARTITION BY toYYYYMMDD(ts)
ORDER BY (host, metric, ts);
```

## Verify Keeper Health

```sql
SELECT *
FROM system.zookeeper
WHERE path = '/clickhouse/tables/01/metrics/replicas';
```

```bash
echo ruok | nc ch1 9181
```

Expected response: `imok`

## Verify All Three Replicas Are in Sync

```sql
SELECT replica_name, is_leader, total_replicas, queue_size
FROM system.replicas
WHERE table = 'metrics';
```

All three replicas should show `total_replicas = 3` and `queue_size = 0`.

## Summary

Three-node ClickHouse replication with embedded Keeper eliminates external ZooKeeper dependencies and tolerates one node failure without losing quorum. Use per-node `server_id` macros in Keeper config and unique `{replica}` macros in cluster config to keep the setup consistent across restarts and replacements.
