# How to Monitor ClickHouse Keeper Health and Metrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Keeper, Monitoring, Metrics, Health Check

Description: Learn how to monitor ClickHouse Keeper health using four-letter commands, Prometheus metrics, and system table queries to detect issues early.

---

ClickHouse Keeper exposes health information through several interfaces: four-letter word commands, Prometheus metrics, and system tables accessible from connected ClickHouse servers. Monitoring these regularly prevents coordination failures from going undetected.

## Four-Letter Commands

Keeper supports ZooKeeper-compatible four-letter commands over the client port:

```bash
# Basic health check
clickhouse-keeper-client -h localhost -p 9181 -q "ruok"
# Returns: imok

# Server statistics
clickhouse-keeper-client -h localhost -p 9181 -q "stat"

# Detailed metrics
clickhouse-keeper-client -h localhost -p 9181 -q "mntr"

# List connected clients
clickhouse-keeper-client -h localhost -p 9181 -q "cons"

# Current configuration
clickhouse-keeper-client -h localhost -p 9181 -q "conf"
```

## Key Metrics from mntr

The `mntr` command returns metrics in key-value format:

```bash
clickhouse-keeper-client -h localhost -p 9181 -q "mntr" | grep -E "latency|connections|state|znodes"
```

```text
zk_avg_latency          5
zk_max_latency          42
zk_min_latency          0
zk_num_alive_connections 8
zk_server_state         leader
zk_znode_count          12847
zk_watch_count          1203
zk_outstanding_requests 0
```

Alert thresholds to consider:
- `zk_avg_latency` above 100ms indicates performance issues
- `zk_outstanding_requests` above 10 indicates saturation
- `zk_num_alive_connections` dropping to 0 indicates ClickHouse servers disconnected

## Prometheus Metrics Export

Enable the Prometheus endpoint in your Keeper config:

```xml
<clickhouse>
    <prometheus>
        <endpoint>/metrics</endpoint>
        <port>9363</port>
        <metrics>true</metrics>
        <asynchronous_metrics>true</asynchronous_metrics>
        <events>true</events>
    </prometheus>
</clickhouse>
```

Key Prometheus metrics:

```text
ClickHouseKeeperResponseTime
ClickHouseKeeperRequestTime
ClickHouseKeeperConnections
ClickHouseKeeperOutstandingRequests
ClickHouseMetrics_KeeperAliveConnections
```

## Monitoring from ClickHouse Server

Query Keeper state directly from a connected ClickHouse server:

```sql
-- Check ZooKeeper/Keeper connectivity
SELECT * FROM system.zookeeper WHERE path = '/';

-- Check session state
SELECT * FROM system.zookeeper_connection;
```

```sql
-- Monitor replication coordination health via Keeper
SELECT
    database,
    table,
    zookeeper_exception,
    total_replicas,
    active_replicas
FROM system.replicas
WHERE length(zookeeper_exception) > 0;
```

## Prometheus Alerting Rules

Create alerts for critical Keeper conditions:

```text
groups:
  - name: clickhouse_keeper
    rules:
      - alert: KeeperHighLatency
        expr: ClickHouseKeeperResponseTime > 0.1
        for: 5m
        annotations:
          summary: "ClickHouse Keeper high response latency"

      - alert: KeeperNoConnections
        expr: ClickHouseMetrics_KeeperAliveConnections == 0
        for: 1m
        annotations:
          summary: "No ClickHouse servers connected to Keeper"
```

## Log-Based Monitoring

Keeper logs to the standard ClickHouse log path. Watch for election events:

```bash
sudo journalctl -u clickhouse-keeper -f | grep -E "leader|election|error|warning"
```

## Disk Usage Monitoring

Monitor log and snapshot storage:

```bash
du -sh /var/lib/clickhouse/coordination/log/
du -sh /var/lib/clickhouse/coordination/snapshots/

# Alert if log directory exceeds threshold
find /var/lib/clickhouse/coordination/log -name "*.log" | wc -l
```

## Summary

Monitor ClickHouse Keeper using `mntr` for real-time metrics, the Prometheus endpoint for time-series alerting, and `system.zookeeper_connection` from ClickHouse servers. Alert on high average latency, outstanding requests exceeding 10, and zero alive connections. Watch Keeper logs for unexpected leader elections which can indicate network instability.
