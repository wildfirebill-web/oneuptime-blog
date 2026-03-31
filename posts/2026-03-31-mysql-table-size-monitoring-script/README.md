# How to Write a MySQL Table Size Monitoring Script

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Monitoring, Script, Database, Performance

Description: Learn how to write a MySQL table size monitoring script using information_schema to alert on tables growing beyond expected thresholds.

---

## Why Table Size Monitoring Matters

Uncontrolled table growth is one of the most common causes of MySQL performance degradation and operational incidents. A table that doubles in size can invalidate query plans that worked fine before, cause index scans to take much longer, and eventually exhaust disk space. Proactive monitoring lets you catch growth trends before they become emergencies.

Common scenarios that cause unexpected table growth:

- Application logs or audit records being written without a retention policy
- Soft-delete patterns that accumulate rows marked as deleted but never purged
- Queue tables where consumers fall behind producers
- Missing partition pruning causing old data to accumulate

## Querying Table Sizes from information_schema

The `information_schema.TABLES` view provides row counts and size estimates for all tables:

```sql
SELECT
  TABLE_SCHEMA AS db,
  TABLE_NAME AS tbl,
  TABLE_ROWS AS estimated_rows,
  ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_mb,
  ROUND(DATA_LENGTH / 1024 / 1024, 2) AS data_mb,
  ROUND(INDEX_LENGTH / 1024 / 1024, 2) AS index_mb,
  ROUND(DATA_FREE / 1024 / 1024, 2) AS free_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'performance_schema', 'mysql', 'sys')
ORDER BY total_mb DESC
LIMIT 20;
```

Note that `TABLE_ROWS` is an estimate for InnoDB tables. For exact counts, run `SELECT COUNT(*)` directly against the table.

## Building the Monitoring Script

The following script checks each table against a configurable size threshold and sends an alert if exceeded:

```bash
#!/bin/bash
# mysql-table-size-monitor.sh
# Usage: ./mysql-table-size-monitor.sh [threshold_mb]

MYSQL_HOST="localhost"
MYSQL_USER="monitor"
MYSQL_PASS="secret"
THRESHOLD_MB="${1:-1024}"   # Default: alert on tables larger than 1 GB
ALERT_EMAIL="dba@example.com"
LOG_FILE="/var/log/mysql-table-sizes.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

QUERY="
SELECT
  TABLE_SCHEMA,
  TABLE_NAME,
  ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS total_mb
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema','performance_schema','mysql','sys')
  AND (DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024 > ${THRESHOLD_MB}
ORDER BY total_mb DESC;
"

RESULTS=$(mysql -h "$MYSQL_HOST" -u "$MYSQL_USER" -p"$MYSQL_PASS" \
  -N -B -e "$QUERY" 2>/dev/null)

if [ -n "$RESULTS" ]; then
  echo "[$DATE] Tables exceeding ${THRESHOLD_MB} MB:" >> "$LOG_FILE"
  echo "$RESULTS" >> "$LOG_FILE"

  # Send alert email
  echo -e "Subject: MySQL Table Size Alert\n\nTables exceeding ${THRESHOLD_MB} MB:\n\n$RESULTS" \
    | sendmail "$ALERT_EMAIL"
else
  echo "[$DATE] No tables exceed ${THRESHOLD_MB} MB." >> "$LOG_FILE"
fi
```

Make the script executable and schedule it via cron:

```bash
chmod +x /usr/local/bin/mysql-table-size-monitor.sh

# Add to crontab - runs at 7 AM daily
crontab -e
# 0 7 * * * /usr/local/bin/mysql-table-size-monitor.sh 1024
```

## Tracking Growth Over Time

To detect accelerating growth, store historical snapshots and compare them:

```sql
CREATE TABLE IF NOT EXISTS monitoring.table_size_history (
  id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  captured_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  db_name     VARCHAR(64) NOT NULL,
  table_name  VARCHAR(64) NOT NULL,
  total_mb    DECIMAL(12, 2) NOT NULL,
  INDEX idx_table_time (db_name, table_name, captured_at)
);
```

Insert a snapshot daily:

```sql
INSERT INTO monitoring.table_size_history (db_name, table_name, total_mb)
SELECT
  TABLE_SCHEMA,
  TABLE_NAME,
  ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2)
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'performance_schema', 'mysql', 'sys');
```

Query week-over-week growth:

```sql
SELECT
  a.db_name,
  a.table_name,
  a.total_mb AS current_mb,
  b.total_mb AS week_ago_mb,
  ROUND(a.total_mb - b.total_mb, 2) AS growth_mb
FROM monitoring.table_size_history a
JOIN monitoring.table_size_history b
  ON a.db_name = b.db_name
  AND a.table_name = b.table_name
  AND b.captured_at BETWEEN DATE_SUB(NOW(), INTERVAL 8 DAY)
                         AND DATE_SUB(NOW(), INTERVAL 6 DAY)
WHERE a.captured_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)
  AND a.total_mb - b.total_mb > 100
ORDER BY growth_mb DESC;
```

## Summary

A MySQL table size monitoring script queries `information_schema.TABLES` for data and index sizes, compares them against thresholds, and sends alerts when tables grow beyond expected bounds. Storing daily snapshots and tracking week-over-week growth helps you catch accelerating growth trends early, giving you time to add retention policies, archival jobs, or capacity before disk pressure becomes critical.
