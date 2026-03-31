# How to Fix "Received timeout" Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Troubleshooting, Timeout, Performance, Configuration

Description: Fix timeout errors in ClickHouse by identifying which timeout setting triggered, extending appropriate limits, and optimizing slow queries.

---

Timeout errors in ClickHouse can occur at multiple levels: the client connection, ClickHouse's internal query execution limits, or network timeouts. Understanding which timeout fired is key to applying the right fix.

## Types of Timeout Errors

ClickHouse has several distinct timeout settings:

```text
receive_timeout          - waiting for data from client
send_timeout             - sending data to client
query_wait_timeout_ms    - waiting in the query queue
max_execution_time       - total query execution time
lock_acquire_timeout     - waiting for table lock
hedged_connection_timeout_ms - distributed query timeout
```

## Identifying the Timeout Type

Check the error message carefully:

```bash
sudo grep -i "timeout" /var/log/clickhouse-server/clickhouse-server.log | tail -20
```

Common error patterns:

```text
"Timeout exceeded: elapsed 300 seconds"   -> max_execution_time
"Timed out waiting for lock"              -> lock_acquire_timeout
"Timeout: closed by timeout"              -> receive_timeout or send_timeout
```

## Fix 1: Increase max_execution_time

For long-running analytical queries:

```sql
-- For a single query
SET max_execution_time = 3600;
SELECT ... long query ...;
```

Or in the user profile:

```xml
<clickhouse>
  <profiles>
    <default>
      <max_execution_time>3600</max_execution_time>
    </default>
    <readonly_user>
      <max_execution_time>600</max_execution_time>
    </readonly_user>
  </profiles>
</clickhouse>
```

## Fix 2: Increase Connection Timeouts

For slow network or large data transfers:

```xml
<clickhouse>
  <receive_timeout>300</receive_timeout>
  <send_timeout>300</send_timeout>
</clickhouse>
```

Or per connection from the client:

```bash
clickhouse-client \
  --receive_timeout=300 \
  --send_timeout=300 \
  --query "SELECT ..."
```

## Fix 3: Distributed Query Timeouts

For queries across shards:

```sql
SET distributed_ddl_task_timeout = 600;
SET receive_timeout = 600;
SET send_timeout = 600;

-- For distributed queries specifically
SET max_execution_time = 0;  -- No limit
```

## Fix 4: Query Optimization

Rather than extending timeouts, optimize the query:

```sql
-- Check what the query is doing
EXPLAIN PIPELINE SELECT ...;

-- Check if indexes are being used
SELECT read_rows, read_bytes, query
FROM system.query_log
WHERE query_id = 'your-query-id'
  AND type = 'QueryFinish';
```

Common fixes:
- Add columns to the ORDER BY key to improve filtering
- Use PREWHERE instead of WHERE for selective conditions
- Avoid SELECT * - select only needed columns
- Add sampling for approximate results

## Fix 5: Lock Timeout

If queries are waiting for locks:

```sql
-- See what queries are holding locks
SELECT query_id, query, elapsed
FROM system.processes
ORDER BY elapsed DESC;
```

Increase the lock acquire timeout:

```xml
<clickhouse>
  <lock_acquire_timeout>120</lock_acquire_timeout>
</clickhouse>
```

## Summary

Timeout errors in ClickHouse require identifying the specific timeout setting that fired - usually `max_execution_time` for slow queries or `receive_timeout`/`send_timeout` for network issues. Increase the relevant timeout setting in the user profile or per-query with SET, and optimize slow queries using EXPLAIN PIPELINE to eliminate timeouts at the root cause.
