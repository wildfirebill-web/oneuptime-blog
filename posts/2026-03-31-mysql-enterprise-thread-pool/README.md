# How to Use MySQL Enterprise Thread Pool

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Enterprise, Thread Pool, Performance

Description: Learn how to configure MySQL Enterprise Thread Pool to improve concurrency at high connection counts and reduce context-switching overhead under load.

---

## What Is MySQL Enterprise Thread Pool?

By default, MySQL creates one thread per connection. At high concurrency this leads to excessive context switching and memory consumption. MySQL Enterprise Thread Pool (available in MySQL Enterprise Edition and also in MySQL Community via the `thread_pool` plugin on some platforms) replaces this one-thread-per-connection model with a pool of worker threads that handle multiple connections, significantly improving throughput at scale.

## Installing the Thread Pool Plugin

```sql
-- Install the plugin
INSTALL PLUGIN thread_pool SONAME 'thread_pool.so';
INSTALL PLUGIN tp_thread_state SONAME 'thread_pool.so';
INSTALL PLUGIN tp_thread_group_state SONAME 'thread_pool.so';
INSTALL PLUGIN tp_thread_group_stats SONAME 'thread_pool.so';

-- Verify
SHOW PLUGINS WHERE Name LIKE 'thread%';
```

Or configure at startup in `my.cnf`:

```text
[mysqld]
plugin-load-add=thread_pool.so
thread_handling=pool-of-threads
```

## Key Configuration Variables

```sql
SHOW VARIABLES LIKE 'thread_pool%';
```

Important variables:

```text
thread_pool_size             - number of thread groups (default: CPU count)
thread_pool_max_active_query_threads - max active threads per group
thread_pool_stall_limit      - milliseconds before a stall is detected
thread_pool_idle_timeout     - seconds before an idle thread exits
thread_pool_high_prio_mode   - how high-priority connections are handled
thread_pool_algorithm        - scheduling algorithm (0=default, 1=high-concurrency)
```

## Tuning Thread Pool Size

Set `thread_pool_size` to the number of CPU cores for CPU-bound workloads:

```sql
-- Set thread pool size to match available CPU cores
SET GLOBAL thread_pool_size = 16;

-- For I/O-bound workloads, you can increase this
SET GLOBAL thread_pool_size = 32;
```

Persist in `my.cnf`:

```text
[mysqld]
thread_pool_size=16
thread_pool_stall_limit=100
```

## Monitoring Thread Pool Activity

```sql
-- Check thread group status
SELECT * FROM information_schema.TP_THREAD_GROUP_STATE\G

-- Check thread-level state
SELECT * FROM information_schema.TP_THREAD_STATE
WHERE TYPE = 'WORKER';

-- Check statistics per thread group
SELECT GROUP_ID, CONNECTIONS_STARTED, QUERIES_EXECUTED,
       THREADS_CREATED, STALLS
FROM information_schema.TP_THREAD_GROUP_STATS
ORDER BY GROUP_ID;
```

## Understanding Stalls

A "stall" occurs when all active threads in a group are blocked (waiting for I/O or a lock) and no queries are making progress. The thread pool responds by creating a new thread to unblock the group:

```sql
-- Lower stall_limit for faster response to blocking queries
SET GLOBAL thread_pool_stall_limit = 50;  -- 50ms

-- Higher stall_limit reduces thread creation overhead for slow queries
SET GLOBAL thread_pool_stall_limit = 500; -- 500ms
```

## High-Priority Connections

Transactions already in progress can be assigned high priority to prevent starvation:

```sql
-- Configure high-priority mode
SET GLOBAL thread_pool_high_prio_mode = 'transactions';

-- This ensures in-flight transactions get preferential scheduling
-- reducing deadlock risk under high concurrency
```

## Comparing One-Thread-Per-Connection vs Thread Pool

```text
One-Thread-Per-Connection:
  - Simple, works well up to ~200-300 connections
  - Each connection holds a dedicated OS thread
  - Memory and context-switch overhead grows linearly

Thread Pool:
  - Efficient above 300+ connections
  - Fixed number of OS threads serves many connections
  - Reduces context switching, improves CPU utilization
  - Better tail latency under burst traffic
```

## Summary

MySQL Enterprise Thread Pool is a high-concurrency optimization that replaces the default one-thread-per-connection model with a managed pool of worker threads. Configure `thread_pool_size` to match your CPU count, tune `thread_pool_stall_limit` based on your query latency profile, and monitor via `information_schema.TP_THREAD_GROUP_STATS` to verify it is reducing stalls and improving throughput in production.
