# How to Monitor MySQL Replication Lag with Alerts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Replication, Lag, Alert, Monitoring

Description: Monitor MySQL replication lag using SHOW SLAVE STATUS, Performance Schema, and Prometheus alerts to detect and respond to replicas falling behind the primary.

---

## Why Replication Lag Matters

Replication lag is the number of seconds a replica is behind the primary. Applications that read from replicas may receive stale data if lag is high. When lag reaches tens of minutes, a failover that promotes the replica can lose significant data. Monitoring lag and alerting early gives you time to investigate and remediate before either scenario occurs.

## Checking Lag with SHOW SLAVE STATUS

The quickest way to check replication lag on a replica:

```sql
SHOW SLAVE STATUS\G
```

Key fields:

```text
Seconds_Behind_Master: 0      -- lag in seconds (NULL = IO thread not connected)
Slave_IO_Running: Yes          -- is the IO thread running?
Slave_SQL_Running: Yes         -- is the SQL thread running?
Last_SQL_Error:                -- error text if SQL thread stopped
```

## Monitoring Lag via Performance Schema

For GTID-based replication, `performance_schema.replication_applier_status_by_worker` provides per-worker lag:

```sql
SELECT CHANNEL_NAME,
       WORKER_ID,
       APPLYING_TRANSACTION,
       LAST_APPLIED_TRANSACTION,
       APPLYING_TRANSACTION_ORIGINAL_COMMIT_TIMESTAMP,
       APPLYING_TRANSACTION_IMMEDIATE_COMMIT_TIMESTAMP,
       TIMESTAMPDIFF(SECOND,
         APPLYING_TRANSACTION_ORIGINAL_COMMIT_TIMESTAMP,
         NOW()) AS lag_seconds
FROM performance_schema.replication_applier_status_by_worker
WHERE APPLYING_TRANSACTION != '';
```

This shows the original commit timestamp of the transaction currently being applied, giving a more accurate lag measurement than `Seconds_Behind_Master` for parallel replication.

## Scripted Lag Check

Create a simple monitoring script:

```bash
#!/bin/bash
THRESHOLD=30
HOST="replica1.db.local"
LAG=$(mysql -h $HOST -u monitor -pmonitor_secret -e \
  "SHOW SLAVE STATUS\G" 2>/dev/null | \
  grep "Seconds_Behind_Master" | awk '{print $2}')

if [ "$LAG" = "NULL" ]; then
  echo "CRITICAL: Replication IO thread not connected on $HOST"
  exit 2
elif [ "$LAG" -gt "$THRESHOLD" ]; then
  echo "WARNING: Replication lag ${LAG}s on $HOST (threshold ${THRESHOLD}s)"
  exit 1
else
  echo "OK: Replication lag ${LAG}s on $HOST"
  exit 0
fi
```

Schedule this with cron or integrate it as a Nagios check.

## Prometheus Alerting Rules for Replication Lag

```yaml
groups:
  - name: mysql-replication
    rules:
      - alert: MySQLReplicationLagWarning
        expr: mysql_slave_status_seconds_behind_master > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Replication lag {{ $value }}s on {{ $labels.instance }}"

      - alert: MySQLReplicationLagCritical
        expr: mysql_slave_status_seconds_behind_master > 60
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Critical replication lag {{ $value }}s on {{ $labels.instance }}"

      - alert: MySQLReplicationThreadDown
        expr: mysql_slave_status_slave_sql_running == 0
          or mysql_slave_status_slave_io_running == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Replication thread stopped on {{ $labels.instance }}"
```

## Common Causes and Remediation

```text
Cause                          Remediation
-----                          -----------
Long-running write on primary  Break large transactions into smaller batches
Missing index on replica        Add index to replica before promoting
Network saturation              Increase binlog compression; use row-based format
Single-threaded SQL applier     Enable parallel replication: slave_parallel_workers=4
```

Enable parallel replication to reduce lag caused by sequential transaction application:

```sql
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL slave_parallel_workers = 4;
```

## Summary

Monitor MySQL replication lag using `SHOW SLAVE STATUS`, Performance Schema applier status tables, or the MySQL Exporter with Prometheus. Set warning alerts at 10-30 seconds and critical alerts at 60 seconds or more. When lag is detected, check for long-running writes on the primary, missing indexes on the replica, and whether parallel replication is enabled to maximize applier throughput.
