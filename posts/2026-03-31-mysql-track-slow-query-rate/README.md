# How to Track MySQL Slow Query Rate

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance, Monitoring, Slow Query, Database

Description: Learn how to track MySQL slow query rate using status variables, slow query log, and Prometheus metrics to catch performance regressions early.

---

## What the Slow Query Rate Tells You

The slow query rate measures how many queries per second are taking longer than `long_query_time` to execute. A rising slow query rate is one of the clearest signals that something in your application or schema has changed for the worse. Common causes include:

- A missing index after a schema change or data growth
- A new feature deploying unoptimized queries
- Table statistics becoming stale, causing the optimizer to choose bad plans
- Lock contention slowing queries that are waiting for row locks

Tracking this metric alongside query throughput lets you calculate the percentage of queries that are slow, which is more actionable than a raw count.

## Enabling the Slow Query Log

Before tracking the rate, make sure slow query logging is enabled:

```sql
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';
SET GLOBAL log_queries_not_using_indexes = 'ON';
```

For persistent configuration, add these to `/etc/mysql/mysql.conf.d/mysqld.cnf`:

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
```

## Reading Slow Query Count from Status Variables

MySQL tracks the cumulative count of slow queries in the `Slow_queries` status variable:

```sql
SHOW GLOBAL STATUS LIKE 'Slow_queries';
```

To compute the rate, sample this value at intervals:

```bash
#!/bin/bash
MYSQL="mysql -u monitor -p'secret' -N -B"

V1=$($MYSQL -e "SHOW GLOBAL STATUS LIKE 'Slow_queries';" | awk '{print $2}')
sleep 60
V2=$($MYSQL -e "SHOW GLOBAL STATUS LIKE 'Slow_queries';" | awk '{print $2}')

RATE=$(( (V2 - V1) / 60 ))
echo "Slow queries per second: $RATE"
```

## Calculating Slow Query Percentage

Raw rate is useful, but percentage of total queries is more meaningful:

```bash
#!/bin/bash
MYSQL="mysql -u monitor -p'secret' -N -B"

read SLOW1 TOTAL1 <<< $(
  $MYSQL -e "
    SELECT
      (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Slow_queries'),
      (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Questions');
  "
)

sleep 60

read SLOW2 TOTAL2 <<< $(
  $MYSQL -e "
    SELECT
      (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Slow_queries'),
      (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Questions');
  "
)

SLOW_DELTA=$(( SLOW2 - SLOW1 ))
TOTAL_DELTA=$(( TOTAL2 - TOTAL1 ))

if [ $TOTAL_DELTA -gt 0 ]; then
  PCT=$(echo "scale=2; $SLOW_DELTA * 100 / $TOTAL_DELTA" | bc)
  echo "Slow query percentage: $PCT%"
fi
```

## Prometheus Alerting for Slow Query Rate

Using `mysqld_exporter`, you can alert on slow query rate with:

```yaml
groups:
  - name: mysql_slow_queries
    rules:
      - alert: MySQLHighSlowQueryRate
        expr: rate(mysql_global_status_slow_queries[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High slow query rate on {{ $labels.instance }}"
          description: "More than 10 slow queries per second averaged over 5 minutes."

      - alert: MySQLSlowQueryPercentageHigh
        expr: >
          rate(mysql_global_status_slow_queries[5m]) /
          rate(mysql_global_status_questions[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "More than 5% of queries are slow on {{ $labels.instance }}"
```

## Analyzing Slow Queries with pt-query-digest

Once you have slow query logging enabled, use `pt-query-digest` to aggregate patterns:

```bash
pt-query-digest /var/log/mysql/slow.log \
  --since '1h ago' \
  --limit 10 \
  --report-format query_report
```

This groups slow queries by fingerprint and ranks them by total query time, helping you identify which query pattern is responsible for the most overhead.

## Summary

Tracking MySQL slow query rate combines reading the `Slow_queries` status variable over time intervals with the slow query log for detailed analysis. Alerting when the rate or percentage of slow queries exceeds a baseline threshold helps detect performance regressions early, before they compound into a user-visible outage.
