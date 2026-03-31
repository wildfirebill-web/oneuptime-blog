# What Is the Slow Query Log in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Slow Query Log, Performance, Query Optimization, Monitoring

Description: The MySQL slow query log records queries that exceed a configurable execution time threshold, making it the primary tool for identifying performance bottlenecks.

---

## Overview

The slow query log is a MySQL log that records SQL statements that take longer than a specified number of seconds to execute. It is the most important tool for ongoing MySQL performance monitoring because it automatically captures problematic queries without the overhead of logging everything. By regularly analyzing the slow query log, database administrators can identify the top candidates for query optimization and index improvements.

## Enabling the Slow Query Log

```sql
-- Enable at runtime
SET GLOBAL slow_query_log = ON;
SET GLOBAL slow_query_log_file = '/var/log/mysql/mysql-slow.log';
SET GLOBAL long_query_time = 1;  -- Log queries taking > 1 second

-- Verify settings
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';
```

To enable persistently in `my.cnf`:

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 1
log_queries_not_using_indexes = 1
```

## Logging Queries Without Indexes

Enable `log_queries_not_using_indexes` to catch full table scans even if they are fast:

```sql
SET GLOBAL log_queries_not_using_indexes = ON;
```

This is useful for finding missing indexes before they become a problem at scale.

## What a Slow Query Log Entry Looks Like

```
# Time: 2026-03-31T10:22:45.123456Z
# User@Host: app_user[app_user] @ localhost []  Id:  1042
# Query_time: 3.145678  Lock_time: 0.000102  Rows_sent: 150  Rows_examined: 2500000
SET timestamp=1743384165;
SELECT * FROM orders WHERE YEAR(created_at) = 2025 AND status = 'shipped';
```

Key fields:
- `Query_time`: total execution time in seconds
- `Lock_time`: time spent waiting for locks
- `Rows_sent`: rows returned to the client
- `Rows_examined`: rows MySQL scanned to produce the result (high values indicate poor indexing)

## Analyzing with mysqldumpslow

The `mysqldumpslow` tool aggregates the slow query log and groups similar queries:

```bash
# Top 10 slowest queries by total time
mysqldumpslow -s t -t 10 /var/log/mysql/mysql-slow.log

# Top 10 queries by count (most frequent slow queries)
mysqldumpslow -s c -t 10 /var/log/mysql/mysql-slow.log

# Filter to queries matching a pattern
mysqldumpslow -s t -t 10 -g "orders" /var/log/mysql/mysql-slow.log
```

## Analyzing with pt-query-digest

The `pt-query-digest` tool from Percona Toolkit provides more detailed analysis:

```bash
pt-query-digest /var/log/mysql/mysql-slow.log > slow_query_report.txt
```

The report shows query fingerprints grouped with percentile latency, call counts, and total time.

## Adjusting the Threshold

Start with a loose threshold and tighten as you fix slow queries:

```sql
-- Capture everything initially to get a baseline
SET GLOBAL long_query_time = 0.1;  -- 100ms

-- After fixing obvious issues, tighten
SET GLOBAL long_query_time = 1;    -- 1 second

-- For optimized systems
SET GLOBAL long_query_time = 0.5;  -- 500ms
```

## Rotating the Log

On Linux, use `logrotate` or flush the log file:

```sql
-- Tell MySQL to close and reopen the slow log (after rotating the file)
FLUSH SLOW LOGS;
```

## Summary

The slow query log is MySQL's built-in mechanism for capturing queries that exceed a time threshold, giving you a focused list of performance bottlenecks without the overhead of logging all queries. It provides execution time, lock wait time, and rows examined to help prioritize optimization work. Combined with `mysqldumpslow` or `pt-query-digest`, it is the standard starting point for MySQL performance tuning.
