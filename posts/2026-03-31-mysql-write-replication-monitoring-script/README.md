# How to Write a MySQL Replication Monitoring Script

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Replication, Monitoring, Script, Bash

Description: Build a replication monitoring script for MySQL that checks replica lag, IO thread status, and SQL thread status, then sends alerts when issues are detected.

---

## Overview

MySQL replication is a cornerstone of high-availability database setups. When replication breaks or falls behind, your read replicas serve stale data and failover becomes risky. A dedicated monitoring script catches these issues before they escalate.

This guide walks you through writing a Bash script that queries replica status, evaluates key metrics, and fires alerts when thresholds are breached.

## What to Monitor

Before writing the script, identify the three critical indicators:

- **Seconds_Behind_Source** - how far the replica lags behind the primary in seconds
- **Replica_IO_Running** - whether the I/O thread is actively fetching binlog events
- **Replica_SQL_Running** - whether the SQL thread is applying events to the local database

A `NULL` lag value combined with stopped threads almost always means replication has broken.

## The Monitoring Script

```bash
#!/usr/bin/env bash

MYSQL_USER="monitor_user"
MYSQL_PASS="your_password"
MYSQL_HOST="127.0.0.1"
MYSQL_PORT="3306"
LAG_THRESHOLD=30
ALERT_EMAIL="dba@example.com"

run_query() {
  mysql -u"${MYSQL_USER}" -p"${MYSQL_PASS}" -h"${MYSQL_HOST}" -P"${MYSQL_PORT}" \
    --silent --skip-column-names -e "$1"
}

check_replication() {
  local status
  status=$(run_query "SHOW REPLICA STATUS\G")

  io_running=$(echo "$status" | grep "Replica_IO_Running" | awk '{print $2}')
  sql_running=$(echo "$status" | grep "Replica_SQL_Running:" | awk '{print $2}')
  lag=$(echo "$status" | grep "Seconds_Behind_Source" | awk '{print $2}')

  if [[ "$io_running" != "Yes" ]]; then
    send_alert "CRITICAL: Replica IO thread is not running (value: ${io_running})"
  fi

  if [[ "$sql_running" != "Yes" ]]; then
    send_alert "CRITICAL: Replica SQL thread is not running (value: ${sql_running})"
  fi

  if [[ "$lag" == "NULL" ]]; then
    send_alert "CRITICAL: Replication lag is NULL - replication may be broken"
  elif [[ "$lag" -gt "$LAG_THRESHOLD" ]]; then
    send_alert "WARNING: Replication lag is ${lag}s (threshold: ${LAG_THRESHOLD}s)"
  else
    echo "OK: Replication is healthy. Lag=${lag}s IO=${io_running} SQL=${sql_running}"
  fi
}

send_alert() {
  local message="$1"
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] ${message}" | tee -a /var/log/mysql_replication.log
  echo "${message}" | mail -s "MySQL Replication Alert on $(hostname)" "${ALERT_EMAIL}"
}

check_replication
```

## Adding a Cron Job

Schedule the script to run every minute using cron:

```bash
chmod +x /usr/local/bin/check_mysql_replication.sh
```

```text
* * * * * /usr/local/bin/check_mysql_replication.sh >> /var/log/mysql_replication_cron.log 2>&1
```

## Creating a Dedicated Monitor User

Never use root for monitoring. Create a minimal-privilege account:

```sql
CREATE USER 'monitor_user'@'127.0.0.1' IDENTIFIED BY 'strong_password';
GRANT REPLICATION CLIENT ON *.* TO 'monitor_user'@'127.0.0.1';
FLUSH PRIVILEGES;
```

The `REPLICATION CLIENT` privilege is the only one needed to run `SHOW REPLICA STATUS`.

## Handling MySQL 8.0 vs Older Versions

MySQL 8.0.22+ renamed commands from `SLAVE` to `REPLICA`. If you support mixed versions, detect the version first:

```bash
MYSQL_VERSION=$(run_query "SELECT @@global.version" | cut -d. -f1,2)

if (( $(echo "$MYSQL_VERSION >= 8.0" | bc -l) )); then
  STATUS_CMD="SHOW REPLICA STATUS\G"
  IO_FIELD="Replica_IO_Running"
  SQL_FIELD="Replica_SQL_Running:"
  LAG_FIELD="Seconds_Behind_Source"
else
  STATUS_CMD="SHOW SLAVE STATUS\G"
  IO_FIELD="Slave_IO_Running"
  SQL_FIELD="Slave_SQL_Running:"
  LAG_FIELD="Seconds_Behind_Master"
fi
```

## Integrating with OneUptime

For production environments, push replication health to OneUptime as a custom heartbeat or metric:

```bash
ONEUPTIME_URL="https://oneuptime.example.com/heartbeat/TOKEN"

if [[ "$io_running" == "Yes" && "$sql_running" == "Yes" && "$lag" -le "$LAG_THRESHOLD" ]]; then
  curl -sf "${ONEUPTIME_URL}" > /dev/null
fi
```

The heartbeat only fires when replication is healthy. If it misses, OneUptime triggers an incident automatically.

## Summary

A MySQL replication monitoring script should check IO thread state, SQL thread state, and lag in seconds on every execution. By running the script via cron and alerting on threshold breaches, you catch replication failures within minutes. For production systems, combine this with a tool like OneUptime to track uptime and escalate incidents through your on-call workflow.
