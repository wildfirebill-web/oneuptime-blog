# How to Fix "ZooKeeper session expired" in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ZooKeeper, Replication, Error, Troubleshooting

Description: Resolve "ZooKeeper session expired" errors in ClickHouse replication by tuning session timeouts, fixing network issues, and monitoring ZooKeeper health.

---

"ZooKeeper session expired" errors in ClickHouse mean the ZooKeeper client lost its session - typically due to network interruption, ZooKeeper overload, or an expired session timeout. When this happens, replicated tables may go temporarily readonly until a new session is established.

## Check ZooKeeper Health

```bash
# Check ZooKeeper status
echo stat | nc zookeeper-host 2181 | grep -E "Mode|Connections|Latency|znodes"

# Check outstanding requests (should be low)
echo mntr | nc zookeeper-host 2181 | grep zk_outstanding_requests
```

From ClickHouse:

```sql
SELECT * FROM system.zookeeper_log
ORDER BY event_time DESC
LIMIT 20;
```

## Increase ZooKeeper Session Timeout

The default session timeout in ClickHouse is 30 seconds. For unstable networks, increase it:

```xml
<!-- /etc/clickhouse-server/config.xml -->
<zookeeper>
  <node>
    <host>zookeeper-host</host>
    <port>2181</port>
  </node>
  <session_timeout_ms>60000</session_timeout_ms>
  <operation_timeout_ms>30000</operation_timeout_ms>
</zookeeper>
```

## Tune ZooKeeper Server-Side Timeouts

On the ZooKeeper server, ensure `maxSessionTimeout` is not too low:

```text
# zoo.cfg
tickTime=2000
minSessionTimeout=4000
maxSessionTimeout=120000
```

## Fix Network Latency Issues

High network latency between ClickHouse and ZooKeeper causes session expirations:

```bash
ping -c 100 zookeeper-host | tail -5
```

ZooKeeper is sensitive to latency spikes. Co-locate ZooKeeper with ClickHouse nodes or use a dedicated low-latency network.

## Migrate to ClickHouse Keeper

ClickHouse Keeper is a ZooKeeper-compatible alternative built into ClickHouse with better performance and fewer session issues:

```xml
<keeper_server>
  <tcp_port>9181</tcp_port>
  <server_id>1</server_id>
  <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
  <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>
  <coordination_settings>
    <operation_timeout_ms>10000</operation_timeout_ms>
    <session_timeout_ms>30000</session_timeout_ms>
  </coordination_settings>
  <raft_configuration>
    <server>
      <id>1</id>
      <hostname>keeper-host</hostname>
      <port>9234</port>
    </server>
  </raft_configuration>
</keeper_server>
```

## Monitor Session Expiration Rate

```sql
SELECT
    toStartOfMinute(event_time) AS minute,
    countIf(type = 'SessionExpired') AS expired
FROM system.zookeeper_log
WHERE event_time > now() - INTERVAL 1 HOUR
GROUP BY minute
ORDER BY minute;
```

## Summary

"ZooKeeper session expired" in ClickHouse is caused by network instability or ZooKeeper overload. Increase `session_timeout_ms` in ClickHouse config and `maxSessionTimeout` in ZooKeeper for short-term relief. Long-term, reduce network latency between services and consider migrating to ClickHouse Keeper for better integration and stability.
