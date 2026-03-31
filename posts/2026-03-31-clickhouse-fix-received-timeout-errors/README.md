# How to Fix 'Received timeout' Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Timeout, Error, Troubleshooting, Performance

Description: Diagnose and fix 'Received timeout' errors in ClickHouse by tuning connection, query, and receive timeout settings for slow queries.

---

ClickHouse "Received timeout" errors indicate a query or connection exceeded the configured time limit before completing. These occur both on the client side (waiting for a response) and on the server side (a distributed sub-query waiting on a remote shard).

## Identify the Timeout Type

ClickHouse has several independent timeout settings:

```sql
SELECT name, value
FROM system.settings
WHERE name IN (
    'receive_timeout',
    'send_timeout',
    'connect_timeout',
    'max_execution_time',
    'distributed_ddl_task_timeout'
);
```

## Increase Query Execution Timeout

For long-running analytical queries, raise `max_execution_time`:

```sql
SET max_execution_time = 3600;  -- 1 hour
SELECT count() FROM large_table WHERE complex_filter = 1;
```

Or set it per-user in `users.xml`:

```xml
<profiles>
  <default>
    <max_execution_time>600</max_execution_time>
  </default>
</profiles>
```

## Fix Distributed Query Timeouts

For distributed queries timing out waiting on shards:

```sql
SET receive_timeout = 300;
SET send_timeout = 300;
SET connect_timeout_with_failover_ms = 1000;
```

In `config.xml` for persistent settings:

```xml
<remote_servers>
  <my_cluster>
    <shard>
      <replica>
        <host>shard1</host>
        <port>9000</port>
        <receive_timeout>300</receive_timeout>
        <send_timeout>300</send_timeout>
      </replica>
    </shard>
  </my_cluster>
</remote_servers>
```

## Fix Client-Side Timeouts

For `clickhouse-client`:

```bash
clickhouse-client \
  --receive_timeout=600 \
  --send_timeout=600 \
  --query "SELECT ..."
```

For HTTP interface via curl:

```bash
curl --max-time 600 \
  "http://localhost:8123/?query=SELECT+count()+FROM+large_table"
```

## Optimize Slow Queries

Often the real fix is query optimization, not raising timeouts:

```sql
-- Check if the query can use an index
EXPLAIN SELECT count() FROM large_table WHERE event_date = today();

-- Add a primary key condition to limit data scanned
SELECT count() FROM large_table
WHERE event_date = today()
  AND user_id > 0;
```

## Summary

ClickHouse timeout errors require identifying whether the limit is on the client, server, or distributed layer. Raise `max_execution_time` for slow analytical queries, increase `receive_timeout` and `send_timeout` for distributed setups, and set client-side flags when using `clickhouse-client` or curl. Always investigate whether query optimization can eliminate the need for higher timeouts.
