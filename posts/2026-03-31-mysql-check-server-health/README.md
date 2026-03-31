# How to Check MySQL Server Health

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Health Check, Monitoring, Administration, Performance

Description: Learn how to comprehensively check MySQL server health by examining connections, InnoDB status, replication lag, error rates, and disk usage.

---

A healthy MySQL server requires more than just being "up." It must be accepting connections, processing queries efficiently, replicating without lag, and operating within resource limits. This guide covers the key health checks you should perform regularly.

## 1. Verify the Server Is Accepting Connections

The simplest liveness check:

```bash
mysqladmin -u root -p ping
```

Output for a healthy server:

```text
mysqld is alive
```

From inside MySQL:

```sql
SELECT 1;
```

## 2. Check Thread and Connection Health

```sql
SHOW GLOBAL STATUS LIKE 'Threads_running';
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Max_used_connections';
SHOW VARIABLES LIKE 'max_connections';
```

Warning signs:
- `Threads_running` consistently above 20-30 indicates a query backlog
- `Max_used_connections` close to `max_connections` indicates connection saturation risk

## 3. Check for Slow Queries

```sql
SHOW GLOBAL STATUS LIKE 'Slow_queries';
```

Compare this to the `Questions` count for a ratio:

```sql
SELECT
  (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Slow_queries') AS slow,
  (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Questions') AS total;
```

## 4. Check InnoDB Buffer Pool Health

```sql
SHOW STATUS LIKE 'Innodb_buffer_pool_read_requests';
SHOW STATUS LIKE 'Innodb_buffer_pool_reads';
```

Calculate disk read ratio:

```sql
SELECT
  ROUND(
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
    NULLIF((SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'), 0)
    * 100, 2
  ) AS disk_read_pct;
```

Above 1% suggests the buffer pool should be larger.

## 5. Check InnoDB Engine Status

The full InnoDB status report shows active transactions, lock waits, and I/O performance:

```sql
SHOW ENGINE INNODB STATUS\G
```

Look for sections titled `TRANSACTIONS` (active lock waits), `FILE I/O` (pending reads/writes), and `BUFFER POOL AND MEMORY`.

## 6. Check Replication Health

On a replica server:

```sql
SHOW REPLICA STATUS\G
```

Key fields:
- `Replica_IO_Running: Yes` - IO thread is running
- `Replica_SQL_Running: Yes` - SQL thread is running
- `Seconds_Behind_Source: 0` - No replication lag

## 7. Check Disk Space

The data directory can grow unexpectedly. Check from the shell:

```bash
du -sh /var/lib/mysql/
df -h /var/lib/mysql
```

Check the size of individual databases:

```sql
SELECT table_schema AS db,
       ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.tables
GROUP BY table_schema
ORDER BY size_mb DESC;
```

## 8. Check for Error Log Warnings

```sql
SELECT logged, prio, error_code, subsystem, data
FROM performance_schema.error_log
WHERE prio IN ('Error', 'Warning')
ORDER BY logged DESC
LIMIT 20;
```

## Summary Healthcheck Query

Combine key metrics into a single diagnostic query:

```sql
SELECT
  (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Threads_running') AS threads_running,
  (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Threads_connected') AS threads_connected,
  (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Slow_queries') AS slow_queries,
  (SELECT VARIABLE_VALUE FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Uptime') AS uptime_secs;
```

## Summary

A complete MySQL health check covers connection saturation, slow query rates, InnoDB buffer pool efficiency, replication lag, and disk space. Run `mysqladmin ping` for quick liveness checks, use `SHOW ENGINE INNODB STATUS` for deep InnoDB diagnostics, and build these queries into a monitoring script or dashboard to catch problems before they affect users.
