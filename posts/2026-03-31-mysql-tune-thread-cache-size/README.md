# How to Tune thread_cache_size for MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Performance, Tuning, Thread, Connection

Description: Tune MySQL thread_cache_size to reduce the overhead of creating new threads for each connection by reusing cached threads from a pool.

---

Every new connection to MySQL requires creating an OS thread to handle it. Thread creation has overhead - typically a few milliseconds per thread. For applications that frequently open and close connections, this overhead adds up. The `thread_cache_size` controls how many idle threads MySQL keeps in a cache for reuse.

## How Thread Caching Works

When a client disconnects, MySQL can either destroy the thread or put it back in the cache. When a new client connects, MySQL checks the cache first. If a cached thread is available, it is reused - skipping the OS-level thread creation cost. The cache size determines the maximum number of threads to keep idle.

Check the current thread cache configuration and activity:

```sql
SHOW VARIABLES LIKE 'thread_cache_size';
SHOW GLOBAL STATUS LIKE 'Threads%';
```

```text
Threads_cached      8
Threads_connected   47
Threads_created     15834
Threads_running     5
```

The key metric is `Threads_created`. If this number grows steadily over time (not just at startup), threads are not being reused from the cache and the cache is too small.

## Calculating the Thread Cache Miss Rate

```sql
SELECT
  variable_value AS threads_created
FROM performance_schema.global_status
WHERE variable_name = 'Threads_created';

-- Also check connections to calculate miss rate
SELECT
  (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Threads_created') AS threads_created,
  (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Connections') AS total_connections,
  ROUND(
    (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Threads_created') /
    (SELECT variable_value FROM performance_schema.global_status WHERE variable_name = 'Connections') * 100,
    2
  ) AS thread_miss_pct;
```

If `thread_miss_pct` is above 1-2%, increasing `thread_cache_size` will help.

## Setting thread_cache_size

MySQL 8.0 auto-configures `thread_cache_size` based on `max_connections`:

```text
thread_cache_size = 8 + (max_connections / 100)
```

For `max_connections = 300`, the default is 11. For high-traffic applications with rapid connection cycling, increase this:

```sql
SET GLOBAL thread_cache_size = 50;
```

In configuration:

```text
[mysqld]
thread_cache_size = 50
```

## Memory Cost of the Thread Cache

Each cached thread holds a stack, typically 256 KB to 1 MB depending on the OS and `thread_stack` setting:

```sql
SHOW VARIABLES LIKE 'thread_stack';
-- Default: 1MB on 64-bit systems

-- Calculate memory for thread cache
SELECT
  @@thread_cache_size * @@thread_stack / 1024 / 1024 AS thread_cache_memory_mb;
```

For `thread_cache_size = 50` and `thread_stack = 1M`, the cache uses up to 50 MB.

## When Thread Cache Does Not Help

Thread caching does not eliminate connection overhead from connection pools that keep long-lived connections. If your application uses a connection pool (Hikari, pgBouncer equivalent, or ProxySQL), threads stay alive and the cache provides minimal benefit. In that case, focus on tuning `max_connections` and connection pool sizes instead.

```sql
-- Check if connections are being reused (high ratio = good pooling)
SELECT
  variable_value AS threads_connected
FROM performance_schema.global_status
WHERE variable_name = 'Threads_connected';

-- If Threads_connected is consistently high and stable, pooling is working
```

## Verifying Cache Effectiveness After Tuning

After increasing `thread_cache_size`, reset the counters and monitor:

```bash
# Flush status counters (requires SUPER privilege)
mysqladmin -uroot -p flush-status

# Check after a few minutes
mysql -uroot -p -e "SHOW GLOBAL STATUS LIKE 'Threads_created';"
```

## Summary

Tune `thread_cache_size` when your application creates many short-lived connections without a connection pool. Monitor `Threads_created` relative to `Connections` - a miss rate above 1-2% indicates insufficient caching. The memory cost is low (roughly 1 MB per cached thread), so setting it to 25-100 for active servers is safe. For applications with proper connection pooling, thread caching provides minimal benefit since connections are long-lived and threads are not frequently recycled.
