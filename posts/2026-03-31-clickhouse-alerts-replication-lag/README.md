# How to Set Up ClickHouse Alerts for Replication Lag

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Replication, Alert, Monitoring, High Availability

Description: Configure replication lag alerts in ClickHouse to detect when replicas fall behind, ensuring data consistency and high availability in clustered deployments.

---

Replication lag alerts are essential for ClickHouse high-availability clusters. When a replica falls behind the leader, read queries on that replica return stale data, and if the replica falls too far behind, catching up requires significant I/O. Early alerting lets you intervene before lag becomes unmanageable.

## Querying Replication Lag

```sql
SELECT
    hostName() AS node,
    database,
    table,
    is_leader,
    absolute_delay AS lag_seconds,
    queue_size,
    active_replicas,
    total_replicas
FROM system.replicas
WHERE absolute_delay > 0
ORDER BY lag_seconds DESC;
```

`absolute_delay` is the most important field - it measures how many seconds behind the most current replica this replica is.

## Setting Alert Thresholds

Create a scheduled check that writes lag events to an alert table:

```sql
CREATE TABLE replication_lag_alerts (
    check_time DateTime DEFAULT now(),
    node String,
    database String,
    table String,
    lag_seconds Float64,
    queue_size UInt32,
    severity LowCardinality(String)
) ENGINE = MergeTree
ORDER BY (check_time, database, table)
TTL check_time + INTERVAL 14 DAY;
```

Populate it from a cron job:

```bash
clickhouse-client --query "
INSERT INTO replication_lag_alerts (node, database, table, lag_seconds, queue_size, severity)
SELECT
    hostName(),
    database,
    table,
    absolute_delay,
    queue_size,
    multiIf(
        absolute_delay > 300, 'critical',
        absolute_delay > 60, 'warning',
        'ok'
    ) AS severity
FROM system.replicas
WHERE absolute_delay > 30
"
```

## Prometheus Alert Rules

```yaml
groups:
  - name: clickhouse_replication
    rules:
      - alert: ClickHouseReplicationLagWarning
        expr: ClickHouseReplicasAbsoluteDelay > 60
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Replication lag {{ $value }}s on {{ $labels.table }}"

      - alert: ClickHouseReplicationLagCritical
        expr: ClickHouseReplicasAbsoluteDelay > 300
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Critical replication lag {{ $value }}s - {{ $labels.instance }}"

      - alert: ClickHouseReplicaInactive
        expr: ClickHouseReplicasTotalReplicas - ClickHouseReplicasActiveReplicas > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Replica inactive on {{ $labels.instance }}"
```

## Checking Queue-Based Lag

Sometimes `absolute_delay` is 0 but the queue is growing, indicating a temporary spike:

```sql
SELECT
    database,
    table,
    count() AS queue_depth,
    min(create_time) AS oldest_task,
    dateDiff('second', min(create_time), now()) AS oldest_task_age_seconds
FROM system.replication_queue
GROUP BY database, table
HAVING oldest_task_age_seconds > 120
ORDER BY oldest_task_age_seconds DESC;
```

## Responding to Lag Alerts

When a replication lag alert fires:

1. Check if the lagging replica is still alive and connected to Keeper
2. Verify network bandwidth is not saturated between nodes
3. Check if a large merge is blocking replication queue processing

```bash
# Force a replica to sync immediately
clickhouse-client --query "SYSTEM SYNC REPLICA mydb.my_table"

# Check if a specific node is catching up
clickhouse-client --host lagging-node --query "
SELECT absolute_delay FROM system.replicas WHERE table = 'my_table'"
```

## Summary

Replication lag alerts protect data consistency in ClickHouse clusters. Set warning alerts at 60 seconds and critical alerts at 300 seconds. Monitor both `absolute_delay` and `queue_size` as complementary signals, and have a runbook ready that includes `SYSTEM SYNC REPLICA` and network diagnostics to resolve lag incidents quickly.
