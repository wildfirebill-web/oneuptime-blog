# How to Monitor ClickHouse Keeper Health and Metrics

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, ClickHouse Keeper, Monitoring, Prometheus, Metrics, Health Check

Description: Learn how to monitor ClickHouse Keeper health using four-letter commands, Prometheus metrics, and system tables to detect issues early.

---

ClickHouse Keeper requires proactive monitoring to catch quorum failures, high latency, and leader instability before they impact your ClickHouse cluster. This guide covers all available monitoring methods.

## Four-Letter Commands

Keeper supports ZooKeeper-compatible four-letter monitoring commands over the client port (9181):

```bash
# Is the server healthy?
echo ruok | nc keeper1 9181
# Expected: imok

# Server statistics
echo stat | nc keeper1 9181

# Detailed monitoring metrics
echo mntr | nc keeper1 9181

# Server configuration
echo conf | nc keeper1 9181

# List client connections
echo cons | nc keeper1 9181
```

Key `mntr` metrics to watch:

```text
zk_avg_latency          -- average request latency in ms (alert if > 50ms)
zk_max_latency          -- max latency spike
zk_outstanding_requests -- queued requests (alert if > 100)
zk_open_file_descriptor_count
leader_uptime           -- how long current leader has been stable
followers               -- should be N-1 for an N-node cluster
```

## Prometheus Metrics

Enable Prometheus in Keeper configuration:

```xml
<prometheus>
  <endpoint>/metrics</endpoint>
  <port>9363</port>
  <metrics>true</metrics>
  <events>true</events>
</prometheus>
```

Key Prometheus metrics:

```text
ClickHouseKeeperAvgLatency       -- average request latency
ClickHouseKeeperMaxLatency       -- max latency
ClickHouseKeeperOutstandingRequests -- request queue depth
ClickHouseKeeperAliveConnections -- active client connections
ClickHouseKeeperEpochsElapsed    -- number of leader elections (alert on increase)
```

## Grafana Dashboard

Create alerts on:

- `ClickHouseKeeperAvgLatency > 100` for sustained periods
- `ClickHouseKeeperOutstandingRequests > 50`
- `ClickHouseKeeperEpochsElapsed` increasing rapidly (leader instability)

## Monitoring via ClickHouse System Tables

From any ClickHouse server connected to Keeper:

```sql
-- Check Keeper connectivity and response time
SELECT *
FROM system.zookeeper_connection;

-- List Keeper nodes and their paths
SELECT name, value, ctime, mtime, version
FROM system.zookeeper
WHERE path = '/clickhouse'
LIMIT 10;
```

## Check Keeper Leader from Client

```bash
echo stat | nc keeper1 9181 | grep -E "^(Mode|Connections|Outstanding)"
```

If the output shows `Mode: standalone` on more than one node, you have a split-brain situation requiring immediate investigation.

## Health Check Script

```bash
#!/bin/bash
for host in keeper1 keeper2 keeper3; do
  result=$(echo ruok | nc "$host" 9181 2>/dev/null)
  if [ "$result" != "imok" ]; then
    echo "ALERT: $host is not healthy"
  else
    echo "OK: $host"
  fi
done
```

## Summary

Monitor ClickHouse Keeper using `ruok`/`mntr` commands for quick health checks, Prometheus metrics for dashboards and alerting, and `system.zookeeper_connection` from ClickHouse for end-to-end connectivity verification. Alert on average latency, outstanding requests, and leader election frequency to catch stability issues early.
