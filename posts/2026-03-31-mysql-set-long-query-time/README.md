# How to Set long_query_time in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Slow Query Log, Performance, Configuration, Optimization

Description: Learn how to set the long_query_time variable in MySQL to control which queries get captured in the slow query log for performance analysis.

---

The `long_query_time` variable defines the threshold in seconds above which MySQL considers a query "slow" and logs it to the slow query log. Tuning this value correctly is the first step toward effective slow query analysis.

## Checking the Current Value

```sql
SHOW VARIABLES LIKE 'long_query_time';
```

The default value is `10` seconds, which is too high for most applications. Most teams set it between `0.5` and `2` seconds depending on their response time requirements.

## Setting long_query_time at Runtime

You can change the threshold without restarting MySQL:

```sql
SET GLOBAL long_query_time = 1;
```

This affects new connections. Existing sessions keep their old threshold unless you also set the session variable:

```sql
SET SESSION long_query_time = 1;
```

## Persisting the Setting in my.cnf

To make the change survive a server restart, add it to `[mysqld]` in `my.cnf`:

```ini
[mysqld]
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
```

Restart MySQL to apply the change:

```bash
sudo systemctl restart mysql
```

## Using Subsecond Precision

`long_query_time` accepts fractional values, allowing subsecond thresholds:

```sql
SET GLOBAL long_query_time = 0.5;
```

This captures any query taking more than 500 milliseconds. For highly optimized applications, even `0.1` seconds can be a useful threshold.

## Logging Queries Without an Index

You can supplement `long_query_time` by also capturing queries that do not use an index, regardless of duration:

```ini
[mysqld]
log_queries_not_using_indexes = ON
log_throttle_queries_not_using_indexes = 10
```

`log_throttle_queries_not_using_indexes` limits the number of such entries per minute to prevent log flooding.

## Checking How Many Slow Queries Have Been Recorded

```sql
SHOW GLOBAL STATUS LIKE 'Slow_queries';
```

This counter increments each time a query exceeds `long_query_time`. Watching this value over time gives you a quick indication of whether query performance is degrading.

## Analyzing the Slow Query Log with pt-query-digest

Once you have accumulated some log data, use Percona Toolkit to summarize it:

```bash
pt-query-digest /var/log/mysql/slow.log > slow_report.txt
```

The report groups queries by fingerprint and shows total execution time, average duration, and the worst offenders.

## Setting a Per-User or Per-Connection Threshold

Individual connections can override the global threshold. This is useful when running a batch job that you know will be slow and you do not want to flood the log:

```sql
-- Suppress logging for this session only
SET SESSION long_query_time = 3600;
```

## Summary

`long_query_time` is the core tuning knob for the MySQL slow query log. Start with a value of `1` second and lower it progressively as you resolve the worst offenders. Use fractional values for latency-sensitive applications, combine it with `log_queries_not_using_indexes` to catch unindexed scans, and analyze the output with `pt-query-digest` to prioritize optimization work.
