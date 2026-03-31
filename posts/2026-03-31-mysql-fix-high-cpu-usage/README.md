# How to Fix High CPU Usage in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance, CPU, Query, Optimization

Description: Fix high CPU usage in MySQL by identifying expensive queries with slow query log and SHOW PROCESSLIST, adding indexes, and tuning thread and sort settings.

---

High CPU usage in MySQL is almost always caused by inefficient queries. The server CPU spikes when MySQL performs full table scans, sorts large datasets in memory, or handles a very high volume of connections. This guide shows how to identify and fix the most common causes.

## Identify What Is Using CPU

First, see what MySQL is doing right now:

```sql
SHOW FULL PROCESSLIST;
```

Look for queries with `State` showing `Copying to tmp table`, `Sorting result`, `Sending data`, or long-running queries in the `Time` column.

For a real-time view:

```bash
# Monitor MySQL process CPU
top -u mysql
htop -u mysql
```

## Enable and Read the Slow Query Log

```sql
SHOW VARIABLES LIKE 'slow_query_log';
SHOW VARIABLES LIKE 'long_query_time';
SHOW VARIABLES LIKE 'slow_query_log_file';

-- Enable slow query log
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- Log queries over 1 second
SET GLOBAL log_queries_not_using_indexes = ON;
```

Analyze with `pt-query-digest`:

```bash
pt-query-digest /var/lib/mysql/slow.log | head -100
```

## Identify CPU-Intensive Queries With performance_schema

```sql
SELECT DIGEST_TEXT, COUNT_STAR, AVG_TIMER_WAIT/1e12 AS avg_sec,
       SUM_CPU_TIME/1e12 AS total_cpu_sec
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_CPU_TIME DESC
LIMIT 10;
```

## Fix: Add Missing Indexes

Full table scans are the leading cause of high CPU. Use `EXPLAIN` to find them:

```sql
EXPLAIN SELECT * FROM orders
WHERE status = 'pending' AND created_at > '2024-01-01';
```

Look for `type: ALL` in the output. Add an index to fix it:

```sql
ALTER TABLE orders ADD INDEX idx_status_created (status, created_at);
```

## Fix: Optimize Sort and Group Operations

Sorting without indexes causes CPU spikes:

```sql
-- Before optimization - sort on un-indexed column
SELECT * FROM products ORDER BY price DESC LIMIT 20;

-- Add index for sorted access
ALTER TABLE products ADD INDEX idx_price (price);
```

Increase `sort_buffer_size` for large sort operations:

```text
[mysqld]
sort_buffer_size = 4M
```

## Fix: Reduce Connection Overhead

Too many short-lived connections cause CPU overhead from authentication and setup:

```sql
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Connections';
```

Use a connection pool in your application. For `max_connections`:

```text
[mysqld]
max_connections = 200
thread_cache_size = 32
```

## Fix: Disable Query Cache (MySQL 5.7)

The query cache causes CPU contention under write-heavy workloads:

```sql
SHOW VARIABLES LIKE 'query_cache_type';

SET GLOBAL query_cache_type = OFF;
SET GLOBAL query_cache_size = 0;
```

## Summary

High MySQL CPU usage is diagnosed through `SHOW PROCESSLIST`, the slow query log, and `performance_schema`. The most impactful fix is adding indexes to eliminate full table scans. Secondary fixes include optimizing sort operations, using connection pooling, and tuning thread settings. Use `EXPLAIN` on every slow query to understand the execution plan before making changes.
