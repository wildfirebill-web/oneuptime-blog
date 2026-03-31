# How to Fix "ZooKeeper session expired" in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ZooKeeper, Replication, Troubleshooting, Distributed Systems

Description: Resolve ClickHouse "ZooKeeper session expired" errors by tuning session timeouts, fixing network issues, and considering migration to ClickHouse Keeper.

---

## Understanding the Error

ClickHouse uses ZooKeeper (or ClickHouse Keeper) to coordinate replicated table operations. When the session between a ClickHouse node and ZooKeeper expires, you will see:

```text
DB::Exception: ZooKeeper session has been expired. (KEEPER_EXCEPTION)
```

This causes replicated tables to enter read-only mode until the connection is re-established.

## Common Causes

- Network latency or packet loss between ClickHouse and ZooKeeper
- ZooKeeper overload (too many watchers, slow disk I/O)
- Misconfigured session timeout (too short)
- ClickHouse node pausing (GC, CPU saturation) causing missed heartbeats

## Diagnosing the Issue

### Check Current ZooKeeper Connection Status

```sql
-- View ZooKeeper connection info
SELECT * FROM system.zookeeper_connection;

-- Check replica status - readonly means ZooKeeper is disconnected
SELECT
    database,
    table,
    replica_name,
    is_readonly,
    zookeeper_path
FROM system.replicas
WHERE is_readonly = 1;
```

### Inspect System Logs

```bash
# Look for session expiry events in the ClickHouse log
grep -i "zookeeper.*expired\|session.*expired" /var/log/clickhouse-server/clickhouse-server.log | tail -50

# Check ZooKeeper server logs
grep -i "session\|expire\|timeout" /var/log/zookeeper/zookeeper.log | tail -50
```

### Monitor ZooKeeper Latency

```bash
# Use the ZooKeeper 4-letter command to check stats
echo "mntr" | nc zookeeper-host 2181 | grep -E "latency|connections|outstanding"

# Check the number of open sessions
echo "srvr" | nc zookeeper-host 2181
```

## Fixing the Issue

### Increase ZooKeeper Session Timeout

In `/etc/clickhouse-server/config.xml`:

```xml
<zookeeper>
    <node index="1">
        <host>zookeeper1.internal</host>
        <port>2181</port>
    </node>
    <!-- Increase from default 30000ms -->
    <session_timeout_ms>60000</session_timeout_ms>
    <operation_timeout_ms>30000</operation_timeout_ms>
</zookeeper>
```

### Optimize ZooKeeper Configuration

In `zoo.cfg` on the ZooKeeper servers:

```text
# Increase tick time (unit for other timeouts)
tickTime=4000

# Allow more time for leader election
initLimit=20
syncLimit=10

# Increase max client connections
maxClientCnxns=500

# Use dedicated SSD for ZooKeeper data
dataDir=/ssd/zookeeper/data
dataLogDir=/ssd/zookeeper/logs
```

### Migrate to ClickHouse Keeper

ClickHouse Keeper is a built-in ZooKeeper replacement that is more tightly integrated and typically more stable:

```xml
<!-- config.xml - use Keeper instead of external ZooKeeper -->
<zookeeper>
    <node index="1">
        <host>clickhouse-node1.internal</host>
        <port>9181</port>
    </node>
    <node index="2">
        <host>clickhouse-node2.internal</host>
        <port>9181</port>
    </node>
    <node index="3">
        <host>clickhouse-node3.internal</host>
        <port>9181</port>
    </node>
</zookeeper>
```

Configure Keeper on each node:

```xml
<!-- keeper_config.xml -->
<keeper_server>
    <tcp_port>9181</tcp_port>
    <server_id>1</server_id>
    <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
    <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>
    <raft_configuration>
        <server>
            <id>1</id>
            <hostname>clickhouse-node1.internal</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>2</id>
            <hostname>clickhouse-node2.internal</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>3</id>
            <hostname>clickhouse-node3.internal</hostname>
            <port>9234</port>
        </server>
    </raft_configuration>
</keeper_server>
```

## Recovery After Session Expiry

When a replica goes read-only after a session expiry, ClickHouse usually reconnects automatically. To force recovery:

```sql
-- Check if the table recovered
SELECT table, is_readonly FROM system.replicas WHERE database = 'analytics';

-- If still readonly, detach and re-attach
DETACH TABLE analytics.events;
ATTACH TABLE analytics.events;
```

## Summary

ZooKeeper session expiry in ClickHouse is caused by network disruptions, ZooKeeper overload, or timeouts that are too short for your environment. Increase `session_timeout_ms` in `config.xml`, ensure ZooKeeper has fast local disk I/O, and monitor latency with the `mntr` 4-letter command. For greenfield deployments, migrate to ClickHouse Keeper to eliminate the external ZooKeeper dependency entirely.
