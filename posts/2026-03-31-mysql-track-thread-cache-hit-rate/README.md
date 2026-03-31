# How to Track MySQL Thread Cache Hit Rate

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Thread, Monitoring

Description: Monitor the MySQL thread cache hit rate using global status variables, tune thread_cache_size, and reduce thread creation overhead.

---

Every new MySQL connection spawns a thread (or reuses a cached one). Thread creation is expensive - it involves OS-level resource allocation. The thread cache stores idle threads so they can be reused for incoming connections without paying the creation cost again.

## How Thread Caching Works

When a connection closes, MySQL checks if the thread cache has room (`thread_cache_size`). If yes, the thread is placed in the cache for reuse. If no, the thread is destroyed. The next new connection either reuses a cached thread (a "hit") or creates a new one (a "miss").

## Measuring Thread Cache Hit Rate

```sql
SELECT VARIABLE_NAME, VARIABLE_VALUE
FROM performance_schema.global_status
WHERE VARIABLE_NAME IN ('Threads_created', 'Connections');
```

- `Connections`: total connection attempts since startup
- `Threads_created`: number of threads created (cache misses)

**Hit rate:**

```sql
SELECT
  ROUND(
    (1 - threads_created / connections) * 100, 2
  ) AS thread_cache_hit_rate_pct
FROM (
  SELECT
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Threads_created') AS threads_created,
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Connections')      AS connections
) t;
```

A healthy hit rate is above 99%. If `Threads_created` is growing rapidly, the cache is undersized.

## Checking Current Thread Cache Settings

```sql
SHOW VARIABLES LIKE 'thread_cache_size';
```

Also check current thread state:

```sql
SELECT VARIABLE_NAME, VARIABLE_VALUE
FROM performance_schema.global_status
WHERE VARIABLE_NAME IN (
  'Threads_cached',
  'Threads_connected',
  'Threads_running',
  'Threads_created'
);
```

## Rate of Thread Creation Over Time

```bash
# Compare Threads_created delta over 60 seconds
T1=$(mysql -u root -p"$MYSQL_PASSWORD" -se \
  "SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME='Threads_created'")
sleep 60
T2=$(mysql -u root -p"$MYSQL_PASSWORD" -se \
  "SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME='Threads_created'")
echo "Threads created per minute: $((T2 - T1))"
```

More than a few per minute on a busy server indicates the cache is too small.

## Tuning thread_cache_size

```sql
-- Check current setting
SHOW VARIABLES LIKE 'thread_cache_size';

-- Increase it dynamically
SET GLOBAL thread_cache_size = 32;
```

Or in `my.cnf`:

```text
[mysqld]
thread_cache_size = 32
```

A rule of thumb: set `thread_cache_size` to approximately 10% of `max_connections`, with a minimum of 8. For a server with `max_connections = 200`, use at least 20.

## Prometheus Metric

With `mysqld_exporter`:

```text
rate(mysql_global_status_threads_created[5m])
```

Alert when this exceeds 1 per second for sustained periods.

## Summary

The thread cache hit rate is derived from `Threads_created` and `Connections` global status counters. A high rate of thread creation adds CPU and memory overhead per connection. Increase `thread_cache_size` to reduce thread creation - a value of 10% of `max_connections` is a practical starting point.
