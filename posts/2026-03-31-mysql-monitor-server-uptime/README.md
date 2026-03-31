# How to Monitor MySQL Server Uptime

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Uptime, Monitoring, Administration, Status

Description: Learn how to monitor MySQL server uptime, track restarts, and calculate availability metrics using built-in status variables and system tools.

---

Monitoring MySQL uptime tells you how long the server has been running since its last restart. This is valuable for detecting unexpected crashes, tracking maintenance windows, and computing availability metrics for SLA reporting.

## Checking Uptime with SHOW STATUS

The simplest way to check how long MySQL has been running:

```sql
SHOW GLOBAL STATUS LIKE 'Uptime';
```

This returns the uptime in seconds. For a human-readable format:

```sql
SELECT
  FLOOR(VARIABLE_VALUE / 86400) AS days,
  FLOOR((VARIABLE_VALUE MOD 86400) / 3600) AS hours,
  FLOOR((VARIABLE_VALUE MOD 3600) / 60) AS minutes,
  (VARIABLE_VALUE MOD 60) AS seconds
FROM performance_schema.global_status
WHERE VARIABLE_NAME = 'Uptime';
```

## Checking Start Time

Calculate the exact timestamp when MySQL started:

```sql
SELECT DATE_SUB(NOW(), INTERVAL VARIABLE_VALUE SECOND) AS started_at
FROM performance_schema.global_status
WHERE VARIABLE_NAME = 'Uptime';
```

## Using mysqladmin from the Shell

Check uptime without connecting to the MySQL client:

```bash
mysqladmin -u root -p status
```

The output includes uptime in seconds along with other quick stats:

```text
Uptime: 864000  Threads: 5  Questions: 128450  ...
```

## Tracking Uptime with a Monitoring Script

A simple shell script to alert if MySQL has restarted recently:

```bash
#!/bin/bash
UPTIME=$(mysql -u root -p"$MYSQL_PASS" -N -e \
  "SELECT VARIABLE_VALUE FROM performance_schema.global_status \
   WHERE VARIABLE_NAME = 'Uptime';")

if [ "$UPTIME" -lt 300 ]; then
  echo "ALERT: MySQL restarted within the last 5 minutes (uptime: ${UPTIME}s)"
  exit 1
fi

echo "MySQL uptime: ${UPTIME} seconds - OK"
exit 0
```

## Uptime Since Flush Status

The `Uptime_since_flush_status` variable tracks how long since the last `FLUSH STATUS` command:

```sql
SHOW GLOBAL STATUS LIKE 'Uptime_since_flush_status';
```

This is useful when you use `FLUSH STATUS` to reset counters and need to know the measurement window for rate calculations.

## Calculating Availability

For SLA calculations, track downtime events and compute availability:

```sql
-- Create a simple restart log table
CREATE TABLE mysql_restart_log (
  id INT AUTO_INCREMENT PRIMARY KEY,
  restarted_at DATETIME NOT NULL,
  reason VARCHAR(255)
);

-- Insert on startup (via init_file or application startup check)
INSERT INTO mysql_restart_log (restarted_at, reason)
VALUES (NOW(), 'Server restart detected');
```

Calculate monthly availability:

```sql
SELECT
  COUNT(*) AS restart_count,
  ROUND(
    (1 - (COUNT(*) * 300 / (30 * 86400.0))) * 100, 4
  ) AS estimated_availability_pct
FROM mysql_restart_log
WHERE restarted_at >= DATE_SUB(NOW(), INTERVAL 30 DAY);
```

## Monitoring Uptime with Prometheus

If you use `mysqld_exporter`, the uptime metric is exposed automatically:

```text
mysql_global_status_uptime
```

Build a Grafana alert to notify when this value resets unexpectedly (a reset indicates a restart).

## Summary

MySQL uptime is available through `SHOW GLOBAL STATUS LIKE 'Uptime'` in seconds, and you can convert it to a start timestamp by subtracting from `NOW()`. Use shell scripts to alert on recent restarts, track restart events in a log table for availability calculations, and expose the `mysql_global_status_uptime` metric through `mysqld_exporter` for production monitoring dashboards.
