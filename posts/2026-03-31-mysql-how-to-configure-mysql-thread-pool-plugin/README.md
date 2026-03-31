# How to Configure MySQL Thread Pool Plugin

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Thread Pool, Performance, Concurrency, Configuration

Description: Learn how to install and configure the MySQL Thread Pool plugin to improve connection handling and reduce overhead under high concurrency workloads.

---

## What Is the MySQL Thread Pool Plugin?

By default, MySQL creates one OS thread per client connection (the "one-thread-per-connection" model). Under high concurrency, this can lead to excessive context switching, memory pressure, and degraded performance.

The Thread Pool plugin replaces this model with a fixed pool of worker threads that handle requests from many connections. It is available in MySQL Enterprise Edition and as part of the Percona Server and MariaDB community distributions.

## How the Thread Pool Works

Instead of spawning a new thread for each connection, the thread pool divides connections into groups (thread groups). Each group has a limited number of worker threads that handle statements from connections in that group. When all workers in a group are busy, new statements wait in a queue rather than consuming OS resources.

This architecture limits the total number of active threads, reducing context switching and memory usage while maintaining high throughput.

## Checking Availability

```sql
-- Check if the plugin is available to load
SELECT * FROM information_schema.PLUGINS WHERE PLUGIN_NAME LIKE 'thread%';
```

On MySQL Enterprise Edition, the plugin library is included. On Percona Server, it is built-in and always active.

## Installing the Plugin (MySQL Enterprise Edition)

```sql
-- Load the plugin at runtime
INSTALL PLUGIN thread_pool SONAME 'thread_pool.so';

-- Verify it is active
SELECT PLUGIN_NAME, PLUGIN_STATUS
FROM information_schema.PLUGINS
WHERE PLUGIN_NAME = 'thread_pool';
```

To make it permanent, add to `my.cnf`:

```ini
[mysqld]
plugin-load-add=thread_pool.so
```

## Key Configuration Variables

```sql
-- Show all thread pool variables
SHOW VARIABLES LIKE 'thread_pool%';
```

| Variable | Default | Description |
|----------|---------|-------------|
| `thread_pool_size` | CPU count | Number of thread groups |
| `thread_pool_max_active_query_threads` | 0 (unlimited) | Max active query threads per group |
| `thread_pool_oversubscribe` | 3 | Extra stalled threads allowed per group |
| `thread_pool_stall_limit` | 6 (x10ms = 60ms) | Time before a query is considered stalled |
| `thread_pool_queue_timeout` | 0 | Timeout for queued statements (ms) |

## Recommended Configuration for OLTP Workloads

```ini
[mysqld]
# Set to match physical CPU count (not hyperthreaded)
thread_pool_size = 8

# Stall limit in 10ms units; 6 = 60ms
# Increase if queries legitimately take longer without being "stalled"
thread_pool_stall_limit = 6

# Allow a few extra threads per group for stalled long queries
thread_pool_oversubscribe = 3

# Optional: limit concurrent queries per group
thread_pool_max_active_query_threads = 4
```

## Monitoring Thread Pool Activity

```sql
-- Thread pool status counters (available with the plugin)
SHOW STATUS LIKE 'Threadpool%';
```

Example output:

```text
+------------------------------+-------+
| Variable_name                | Value |
+------------------------------+-------+
| Threadpool_idle_threads      | 12    |
| Threadpool_threads           | 16    |
| Threadpool_stall_limit       | 6     |
| Threadpool_queued_queries    | 0     |
+------------------------------+-------+
```

Also check for stalled and waiting threads:

```sql
SELECT * FROM information_schema.TP_THREAD_GROUP_STATS\G
SELECT * FROM information_schema.TP_THREAD_STATE\G
```

## Percona Server Thread Pool (Community Alternative)

Percona Server for MySQL includes a high-performance thread pool built-in without requiring an Enterprise license. Configuration is the same as above using the `thread_pool_*` variables.

```bash
# Verify Percona Server is running
mysql -e "SELECT @@version, @@version_comment;"
```

```sql
-- Percona-specific: priority connection queues
SET GLOBAL thread_pool_high_prio_mode = 'statements';
SET GLOBAL thread_pool_high_prio_tickets = 4294967295;
```

## When to Use the Thread Pool

The thread pool provides the most benefit when:

- You have many (hundreds or thousands) concurrent client connections.
- Your workload has many short-lived queries (OLTP).
- You observe high context-switching or CPU overhead due to thread count.

For workloads with few connections and long-running analytical queries, the default thread model may be sufficient.

## Summary

The MySQL Thread Pool plugin improves concurrency handling by replacing the one-thread-per-connection model with a fixed pool of worker threads organized into groups. Configuring `thread_pool_size` to match your CPU count and tuning `thread_pool_stall_limit` for your query duration profile are the key steps. Monitor `Threadpool_*` status variables and `TP_THREAD_GROUP_STATS` to validate that the thread pool is working effectively for your workload.
