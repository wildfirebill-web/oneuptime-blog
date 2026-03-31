# How to Monitor Replication Lag in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Replication, Monitoring, Replication Lag, Performance

Description: Learn multiple ways to monitor MySQL replication lag using SHOW REPLICA STATUS, Performance Schema, and external monitoring tools.

---

## Why Replication Lag Matters

Replication lag is the delay between when a transaction commits on the primary and when it is applied on the replica. High lag means:

- Read queries on the replica return stale data
- Failover may result in data loss
- Backups taken from the replica may be inconsistent

## Method 1 - SHOW REPLICA STATUS

The simplest way to check lag:

```sql
SHOW REPLICA STATUS\G
```

Look at:

```text
Seconds_Behind_Source: 42
```

This value is the difference between the current time on the replica and the timestamp in the binary log event being processed. It is `NULL` if the SQL thread is stopped.

**Limitation:** This metric can be misleading when the replica has caught up but then sits idle. It only updates when events are being processed.

## Method 2 - Performance Schema (More Accurate)

```sql
SELECT
  WORKER_ID,
  LAST_APPLIED_TRANSACTION,
  TIMESTAMPDIFF(
    SECOND,
    LAST_APPLIED_TRANSACTION_ORIGINAL_COMMIT_TIMESTAMP,
    NOW()
  ) AS lag_seconds
FROM performance_schema.replication_applier_status_by_worker;
```

This calculates lag based on actual transaction commit timestamps, which is more accurate than `Seconds_Behind_Source`.

## Method 3 - Heartbeat Table (Custom Solution)

Create a heartbeat table on the primary:

```sql
CREATE TABLE heartbeat (
  server_id INT NOT NULL PRIMARY KEY,
  ts        TIMESTAMP(6) NOT NULL
);
```

Update it periodically with a cron job or event:

```sql
INSERT INTO heartbeat VALUES (@@server_id, NOW(6))
ON DUPLICATE KEY UPDATE ts = NOW(6);
```

On the replica, measure the difference:

```sql
SELECT TIMESTAMPDIFF(MICROSECOND, ts, NOW(6)) / 1000000 AS lag_seconds
FROM heartbeat
WHERE server_id = 1;
```

## Method 4 - pt-heartbeat (Percona Toolkit)

Percona Toolkit's `pt-heartbeat` automates the heartbeat approach:

```bash
# On the primary - start the heartbeat writer
pt-heartbeat --host=primary --database=mydb --update --daemonize

# On the replica - check lag
pt-heartbeat --host=replica --database=mydb --monitor
```

Output:

```text
0.00s [  0.00s,  0.00s,  0.00s ]
```

## Alerting on Lag Thresholds

You can build a simple script to alert when lag exceeds a threshold:

```bash
#!/bin/bash
LAG=$(mysql -u monitor -ppassword -se "SHOW REPLICA STATUS\G" | grep "Seconds_Behind_Source" | awk '{print $2}')
THRESHOLD=30

if [ "$LAG" -gt "$THRESHOLD" ] 2>/dev/null; then
  echo "ALERT: Replication lag is ${LAG}s (threshold: ${THRESHOLD}s)"
fi
```

## Monitoring with Prometheus and mysqld_exporter

If you use Prometheus, `mysqld_exporter` exposes replication lag as a metric:

```text
mysql_slave_status_seconds_behind_master
```

A simple alert rule in Prometheus:

```yaml
- alert: MySQLReplicationLagHigh
  expr: mysql_slave_status_seconds_behind_master > 30
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "MySQL replication lag is {{ $value }}s"
```

## Summary

The quickest way to check replication lag is `SHOW REPLICA STATUS\G` and reading `Seconds_Behind_Source`. For more accuracy, query `performance_schema.replication_applier_status_by_worker`. For production monitoring at scale, use pt-heartbeat or a metrics exporter like `mysqld_exporter` with alerting thresholds.
