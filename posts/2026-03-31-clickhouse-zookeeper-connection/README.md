# How to Configure ClickHouse ZooKeeper Connection

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Database, ZooKeeper, Configuration, Cluster, Replication

Description: Learn how to configure ClickHouse ZooKeeper and ClickHouse Keeper connections for ReplicatedMergeTree tables, DDL replication, and distributed coordination.

---

ClickHouse uses Apache ZooKeeper (or its native replacement ClickHouse Keeper) for distributed coordination. This coordination layer is required for ReplicatedMergeTree tables, `ON CLUSTER` DDL operations, and distributed INSERT deduplication. Without a correct ZooKeeper configuration, replication will not start and clustered DDL will fail.

## What ZooKeeper Does for ClickHouse

- **ReplicatedMergeTree coordination** - replicas register themselves, track which data parts each replica has, and coordinate merges so only one replica performs each merge.
- **DDL replication** - `CREATE TABLE ON CLUSTER`, `ALTER TABLE ON CLUSTER`, and other DDL commands are queued in ZooKeeper so all nodes in the cluster execute them.
- **INSERT deduplication** - ZooKeeper stores block checksums for a configurable window so that retried inserts do not produce duplicate data.

## Connecting to an External ZooKeeper Ensemble

Add the `<zookeeper>` section to `/etc/clickhouse-server/config.xml` or a file in `/etc/clickhouse-server/config.d/`:

```xml
<clickhouse>
    <zookeeper>
        <node>
            <host>zk-01.internal</host>
            <port>2181</port>
        </node>
        <node>
            <host>zk-02.internal</host>
            <port>2181</port>
        </node>
        <node>
            <host>zk-03.internal</host>
            <port>2181</port>
        </node>

        <!-- Session timeout in milliseconds -->
        <session_timeout_ms>30000</session_timeout_ms>

        <!-- Operation timeout in milliseconds -->
        <operation_timeout_ms>10000</operation_timeout_ms>

        <!-- Optional: ZooKeeper root path for this ClickHouse installation -->
        <!-- Useful when sharing a ZooKeeper ensemble between multiple ClickHouse clusters -->
        <root>/clickhouse_prod</root>

        <!-- Optional: identity for ZooKeeper ACLs -->
        <!-- <identity>clickhouse:my_secret</identity> -->
    </zookeeper>
</clickhouse>
```

## Using ClickHouse Keeper Instead of ZooKeeper

ClickHouse Keeper is a built-in ZooKeeper-compatible service that runs inside the ClickHouse process itself. It removes the need to maintain a separate ZooKeeper ensemble and generally offers lower latency.

### Embedded Keeper Configuration

Enable Keeper on dedicated Keeper nodes (or on all ClickHouse nodes for small clusters):

```xml
<clickhouse>
    <keeper_server>
        <!-- Port for the Keeper protocol -->
        <tcp_port>9181</tcp_port>

        <!-- Unique ID for this Keeper node - must differ on each server -->
        <server_id>1</server_id>

        <!-- Where Keeper stores its logs and snapshots -->
        <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
        <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>

        <!-- Raft coordination settings -->
        <coordination_settings>
            <operation_timeout_ms>10000</operation_timeout_ms>
            <session_timeout_ms>30000</session_timeout_ms>
            <raft_logs_level>warning</raft_logs_level>
        </coordination_settings>

        <!-- Define the Keeper cluster (Raft quorum) -->
        <raft_configuration>
            <server>
                <id>1</id>
                <hostname>ch-node-01.internal</hostname>
                <port>9444</port>  <!-- Internal Raft port -->
            </server>
            <server>
                <id>2</id>
                <hostname>ch-node-02.internal</hostname>
                <port>9444</port>
            </server>
            <server>
                <id>3</id>
                <hostname>ch-node-03.internal</hostname>
                <port>9444</port>
            </server>
        </raft_configuration>
    </keeper_server>
</clickhouse>
```

### Connecting ClickHouse to Its Keeper

On all ClickHouse nodes that use this Keeper ensemble:

```xml
<clickhouse>
    <zookeeper>
        <node>
            <host>ch-node-01.internal</host>
            <port>9181</port>
        </node>
        <node>
            <host>ch-node-02.internal</host>
            <port>9181</port>
        </node>
        <node>
            <host>ch-node-03.internal</host>
            <port>9181</port>
        </node>
    </zookeeper>
</clickhouse>
```

Even though the connection config looks the same as for ZooKeeper, ClickHouse is connecting to Keeper's ZooKeeper-compatible port.

## Macros for Shard and Replica Identity

ClickHouse uses macros to construct unique ZooKeeper paths for each replica. Define macros on each node:

```xml
<!-- On ch-node-01 (shard 1, replica 1) -->
<clickhouse>
    <macros>
        <cluster>production_cluster</cluster>
        <shard>01</shard>
        <replica>ch-node-01</replica>
    </macros>
</clickhouse>
```

```xml
<!-- On ch-node-02 (shard 1, replica 2) -->
<clickhouse>
    <macros>
        <cluster>production_cluster</cluster>
        <shard>01</shard>
        <replica>ch-node-02</replica>
    </macros>
</clickhouse>
```

Use these macros when creating replicated tables:

```sql
CREATE TABLE events_local
(
    event_date  Date,
    user_id     UInt64,
    event_type  String
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{cluster}/{shard}/events',  -- ZooKeeper path, unique per shard
    '{replica}'                                      -- Unique per replica
)
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id);
```

## Verifying ZooKeeper Connectivity

```sql
-- Check ZooKeeper session status
SELECT *
FROM system.zookeeper_connection;
```

```sql
-- Browse the ZooKeeper tree used by ClickHouse
SELECT
    name,
    value,
    czxid,
    mzxid,
    ctime,
    mtime,
    numChildren
FROM system.zookeeper
WHERE path = '/clickhouse/tables'
ORDER BY name;
```

## Checking Replication Status

```sql
-- Overall replication health
SELECT
    database,
    table,
    engine,
    is_leader,
    is_readonly,
    is_session_expired,
    future_parts,
    parts_to_check,
    queue_size,
    inserts_in_queue,
    merges_in_queue,
    last_queue_update,
    zookeeper_exception
FROM system.replicas
WHERE zookeeper_exception != ''
   OR is_session_expired = 1
   OR is_readonly = 1;
```

```sql
-- Replication queue: pending operations
SELECT
    database,
    table,
    type,
    source_replica,
    new_part_name,
    create_time,
    required_quorum,
    is_detach,
    last_exception
FROM system.replication_queue
ORDER BY create_time
LIMIT 20;
```

## Tuning ZooKeeper Session Parameters

```xml
<clickhouse>
    <zookeeper>
        <node>
            <host>zk-01.internal</host>
            <port>2181</port>
        </node>

        <!-- Increase for slow networks or large clusters -->
        <session_timeout_ms>60000</session_timeout_ms>

        <!-- Reduce for fast fail-over detection -->
        <operation_timeout_ms>5000</operation_timeout_ms>

        <!-- Retry connecting this many times before reporting an error -->
        <connection_retry_count>3</connection_retry_count>

        <!-- Wait this long between connection retries -->
        <connection_retry_wait_ms>1000</connection_retry_wait_ms>
    </zookeeper>
</clickhouse>
```

## Common Errors and Fixes

```text
Error: No connection to ZooKeeper
```

Check that the ZooKeeper/Keeper nodes are reachable from the ClickHouse server and that the ports are not blocked by firewall rules.

```bash
# Test ZooKeeper connectivity from the ClickHouse host
echo ruok | nc zk-01.internal 2181
# Expected response: imok
```

```text
Error: ZooKeeper session expired
```

Increase `<session_timeout_ms>`. This often happens when ZooKeeper is under load or when there are network hiccups.

```text
Error: Table is in read-only mode
```

A replica enters read-only mode when it cannot reach ZooKeeper. Restore connectivity or increase session/operation timeouts.

## Conclusion

ZooKeeper (or ClickHouse Keeper) is the coordination backbone of any replicated ClickHouse deployment. Configure the `<zookeeper>` section with all nodes of your ensemble, define unique `<macros>` for shard and replica identity on each server, and use those macros in ReplicatedMergeTree paths to guarantee unique coordination paths. Monitor replication health via `system.replicas` and `system.replication_queue`, and use ClickHouse Keeper for new deployments to eliminate the operational overhead of maintaining a separate ZooKeeper ensemble.
