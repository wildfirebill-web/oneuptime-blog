# What Is ClickHouse Keeper and Why You Need It

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ClickHouse Keeper, Replication, Distributed System, ZooKeeper

Description: Learn what ClickHouse Keeper is, how it replaces ZooKeeper for ClickHouse replication coordination, and how to deploy and configure it for production clusters.

## Introduction

ClickHouse clusters using `ReplicatedMergeTree` tables require a coordination service to manage replication metadata. This coordination service tracks which data parts each replica has, coordinates merges across replicas, and handles leader election.

Historically, this role was filled by Apache ZooKeeper - an external system that ClickHouse operators had to deploy and maintain separately. **ClickHouse Keeper** is a built-in alternative that implements the same ZooKeeper protocol but is designed specifically for ClickHouse's coordination needs. It is simpler to deploy, performs better for ClickHouse workloads, and eliminates the dependency on a Java-based external service.

## What ClickHouse Keeper Does

ClickHouse Keeper maintains a distributed, consistent key-value store that ClickHouse replicas use to coordinate:

1. **Replication log** - tracks which operations (inserts, merges, mutations) each replica has completed
2. **Part tracking** - records which data parts each replica currently holds
3. **Leader election** - determines which replica is the current leader for each table
4. **Distributed DDL** - coordinates schema changes across all nodes in a cluster
5. **Distributed table coordination** - helps `Distributed` tables track which shards are available

## ZooKeeper vs ClickHouse Keeper

| Aspect | ZooKeeper | ClickHouse Keeper |
|---|---|---|
| Language | Java | C++ |
| Deployment | Separate cluster | Embedded in ClickHouse binary |
| Protocol | ZooKeeper protocol | ZooKeeper-compatible protocol |
| Performance | Good | Better for ClickHouse workloads |
| Consistency model | ZAB protocol | Raft consensus |
| Configuration complexity | High | Lower |
| Operational overhead | Requires JVM tuning | None |

ClickHouse Keeper uses **Raft consensus** instead of ZooKeeper's ZAB protocol. Raft is generally considered easier to understand and has well-tested implementations. The Raft-based Keeper shows lower write latency and better throughput for ClickHouse's coordination patterns.

## Deployment Options

### Option 1: Embedded in ClickHouse

You can run ClickHouse Keeper as part of the same process as ClickHouse Server. This is simpler for small deployments:

```xml
<!-- In config.xml or a file in config.d/ -->
<keeper_server>
    <tcp_port>9181</tcp_port>
    <server_id>1</server_id>
    <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
    <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>

    <coordination_settings>
        <operation_timeout_ms>10000</operation_timeout_ms>
        <session_timeout_ms>30000</session_timeout_ms>
        <raft_logs_level>warning</raft_logs_level>
    </coordination_settings>

    <raft_configuration>
        <server>
            <id>1</id>
            <hostname>ch-node-01</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>2</id>
            <hostname>ch-node-02</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>3</id>
            <hostname>ch-node-03</hostname>
            <port>9234</port>
        </server>
    </raft_configuration>
</keeper_server>
```

### Option 2: Standalone ClickHouse Keeper

For production, run Keeper on dedicated servers separate from ClickHouse data nodes. This prevents Keeper resource usage from affecting query performance:

```bash
# Install ClickHouse Keeper
apt-get install clickhouse-keeper

# Configure /etc/clickhouse-keeper/keeper_config.xml
```

```xml
<!-- /etc/clickhouse-keeper/keeper_config.xml -->
<clickhouse>
    <logger>
        <level>warning</level>
        <log>/var/log/clickhouse-keeper/clickhouse-keeper.log</log>
        <errorlog>/var/log/clickhouse-keeper/clickhouse-keeper.err.log</errorlog>
    </logger>

    <listen_host>0.0.0.0</listen_host>

    <keeper_server>
        <tcp_port>9181</tcp_port>
        <server_id>1</server_id>
        <log_storage_path>/var/lib/clickhouse-keeper/coordination/log</log_storage_path>
        <snapshot_storage_path>/var/lib/clickhouse-keeper/coordination/snapshots</snapshot_storage_path>

        <raft_configuration>
            <server>
                <id>1</id>
                <hostname>keeper-01</hostname>
                <port>9234</port>
            </server>
            <server>
                <id>2</id>
                <hostname>keeper-02</hostname>
                <port>9234</port>
            </server>
            <server>
                <id>3</id>
                <hostname>keeper-03</hostname>
                <port>9234</port>
            </server>
        </raft_configuration>
    </keeper_server>
</clickhouse>
```

## Connecting ClickHouse to Keeper

Tell ClickHouse servers where to find the Keeper ensemble by adding a `zookeeper` section to `config.xml`:

```xml
<zookeeper>
    <node>
        <host>keeper-01</host>
        <port>9181</port>
    </node>
    <node>
        <host>keeper-02</host>
        <port>9181</port>
    </node>
    <node>
        <host>keeper-03</host>
        <port>9181</port>
    </node>
    <session_timeout_ms>30000</session_timeout_ms>
    <operation_timeout_ms>10000</operation_timeout_ms>
</zookeeper>
```

The `zookeeper` section name is kept for backward compatibility even when using ClickHouse Keeper (which is ZooKeeper-protocol compatible).

## Cluster Size Recommendations

Raft requires a majority quorum to operate. The Keeper cluster must have more than half of its nodes alive to accept writes:

| Keeper nodes | Fault tolerance | Can tolerate |
|---|---|---|
| 1 | None | 0 failures |
| 3 | 1 node | 1 failure |
| 5 | 2 nodes | 2 failures |
| 7 | 3 nodes | 3 failures |

For production, 3 nodes (tolerating 1 failure) is the minimum recommended configuration. 5 nodes is common for high-availability deployments.

## Monitoring ClickHouse Keeper

Check Keeper health and statistics using the `system.zookeeper` table and the `clickhouse-keeper-client` tool:

```sql
-- List all root-level keys in Keeper
SELECT path, name, data_length, num_children
FROM system.zookeeper
WHERE path = '/'
ORDER BY name;
```

```sql
-- Check replication coordination data
SELECT path, name, data_length
FROM system.zookeeper
WHERE path = '/clickhouse/tables'
ORDER BY name;
```

```bash
# Use the keeper client tool to check 4-letter commands
echo "ruok" | nc keeper-01 9181   # Should return "imok"
echo "stat" | nc keeper-01 9181   # Keeper statistics
echo "mntr" | nc keeper-01 9181   # Detailed metrics
```

Check Keeper status from within ClickHouse:

```sql
SELECT *
FROM system.zookeeper_connection;
```

## Keeper Metrics in system.metrics

```sql
SELECT metric, value, description
FROM system.metrics
WHERE metric LIKE '%Keeper%'
   OR metric LIKE '%ZooKeeper%';
```

Important metrics:
- `ZooKeeperRequest` - active ZooKeeper requests
- `ZooKeeperWatch` - active ZooKeeper watches
- `ZooKeeperSession` - active ZooKeeper sessions

## What Happens When Keeper Is Unavailable

If ClickHouse Keeper becomes unavailable:

- **Reads from replicated tables** - still work normally (replicas serve reads independently)
- **Writes to replicated tables** - blocked until Keeper is restored
- **Merges** - paused (cannot coordinate cross-replica merge assignments)
- **DDL operations** - blocked
- **New replica initialization** - blocked

This is why Keeper high availability (minimum 3 nodes) is critical for production deployments that use `ReplicatedMergeTree`.

## Migrating from ZooKeeper to ClickHouse Keeper

The migration is zero-downtime if done carefully:

```text
1. Deploy a ClickHouse Keeper ensemble (3+ nodes)
2. Configure ClickHouse to point to Keeper (same zookeeper config section)
3. Use clickhouse-keeper-converter to copy ZooKeeper state to Keeper
4. Verify replication is healthy
5. Decommission ZooKeeper
```

```bash
# Convert ZooKeeper data to Keeper format
clickhouse-keeper-converter \
    --zookeeper-logs-dir /var/lib/zookeeper/version-2 \
    --zookeeper-snapshots-dir /var/lib/zookeeper/version-2 \
    --output-dir /var/lib/clickhouse-keeper/coordination/snapshots
```

## Conclusion

ClickHouse Keeper is the recommended coordination service for ClickHouse clusters. It replaces Apache ZooKeeper with a purpose-built, C++-based implementation that is simpler to operate and performs better for ClickHouse workloads. A 3-node Keeper ensemble is the minimum for production fault tolerance. For large clusters with high coordination traffic, deploying Keeper on dedicated servers prevents resource contention with ClickHouse query workloads.

**Related Reading:**

- [What Is the Difference Between ReplicatedMergeTree and MergeTree](https://oneuptime.com/blog/post/2026-03-31-clickhouse-replicated-vs-mergetree/view)
- [What Is Distributed Table Engine and How It Routes Queries](https://oneuptime.com/blog/post/2026-03-31-clickhouse-distributed-table-engine/view)
- [How to Use ClickHouse Cloud vs Self-Hosted ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-cloud-vs-self-hosted/view)
