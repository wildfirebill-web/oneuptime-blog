# How to Use remote() and remoteSecure() Table Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Table Function, remote, remoteSecure, Distributed Query

Description: Learn how to use remote() and remoteSecure() table functions in ClickHouse to query data from other ClickHouse servers in real time without setting up a Distributed table.

---

The `remote()` and `remoteSecure()` table functions allow you to execute queries against tables on remote ClickHouse servers directly from a query, without needing to define a Distributed table in advance. `remote()` connects over an unencrypted TCP connection while `remoteSecure()` uses TLS. These functions are useful for ad hoc cross-server queries, data migrations, and federated analytics.

## How remote() Works

```mermaid
graph LR
    A[Local ClickHouse] -->|remote()| B[Remote ClickHouse Server]
    B --> C[Query Result]
    C --> A
    A -->|remoteSecure()| D[Remote ClickHouse - TLS]
    D --> C
```

ClickHouse sends the query to the remote server, executes it there, and streams the result back to the local server. Only the result rows are transferred, not the full table scan traffic.

## Syntax

```sql
remote('host:port', db, table, 'user', 'password')
remote('host:port', db.table, 'user', 'password')
remote('host:port', db, table)
remoteSecure('host:port', db, table, 'user', 'password')
```

Parameters:
- `host:port` - remote server address; port defaults to 9000 for `remote()` and 9440 for `remoteSecure()`
- `db` - database name on the remote server
- `table` - table name on the remote server
- `user` / `password` - optional credentials (defaults to the current session user)

## Basic Usage

### Query a Remote Table

```sql
SELECT count()
FROM remote('other-clickhouse-server:9000', 'analytics', 'events');
```

### Retrieve Rows from Remote Server

```sql
SELECT *
FROM remote('192.168.1.10', 'mydb', 'orders', 'reader', 'secret')
WHERE order_date >= today() - 7
LIMIT 100;
```

### Use remoteSecure() for TLS Connections

```sql
SELECT sum(revenue)
FROM remoteSecure('secure-server.example.com:9440', 'sales', 'transactions')
WHERE status = 'completed';
```

## Complete Working Example

This example demonstrates creating a local table, inserting data locally, and then querying it from a "remote" context (using localhost to simulate the pattern).

```sql
-- On the remote server (or simulate with localhost):
CREATE TABLE remote_sales
(
    sale_id    UInt64,
    product    String,
    amount     Float64,
    sale_date  Date
)
ENGINE = MergeTree()
ORDER BY (sale_date, sale_id);

INSERT INTO remote_sales VALUES
    (1, 'Widget A', 29.99, '2026-03-01'),
    (2, 'Widget B', 49.99, '2026-03-02'),
    (3, 'Widget A', 29.99, '2026-03-03'),
    (4, 'Widget C', 99.99, '2026-03-04'),
    (5, 'Widget B', 49.99, '2026-03-05');

-- Query from the local server using remote():
SELECT
    product,
    count()        AS sales_count,
    sum(amount)    AS total_revenue
FROM remote('localhost', 'default', 'remote_sales')
WHERE sale_date >= '2026-03-01'
GROUP BY product
ORDER BY total_revenue DESC;
```

```text
product  | sales_count | total_revenue
---------+-------------+--------------
Widget C | 1           | 99.99
Widget B | 2           | 99.98
Widget A | 2           | 59.98
```

## Querying Multiple Remote Servers

You can pass multiple hosts as a comma-separated list or use sharding notation. ClickHouse sends the query to each host and unions the results.

```sql
-- Query two shards and aggregate the result
SELECT
    event_type,
    sum(event_count) AS total
FROM remote('shard1:9000,shard2:9000', 'analytics', 'events_summary')
GROUP BY event_type
ORDER BY total DESC;
```

### Shard Range Notation

ClickHouse supports a shorthand for ranges of shards:

```sql
-- Queries shard01, shard02, shard03 in parallel
SELECT count()
FROM remote('shard{01..03}:9000', 'db', 'table');
```

## Inserting Data from a Remote Server

Use `remote()` as the source in an `INSERT INTO ... SELECT` to migrate or copy data.

```sql
-- Copy last 7 days of data from a remote server to local
INSERT INTO local_events
SELECT *
FROM remote('source-server:9000', 'analytics', 'events')
WHERE event_date >= today() - 7;
```

## Performance Considerations

- Queries executed via `remote()` run on the remote server. Only the result is sent back.
- Use `WHERE` clauses to reduce the amount of data transferred.
- For repeated cross-server queries, consider creating a `Distributed` table instead, which provides better integration with ClickHouse's query planner.
- Avoid using `remote()` in high-frequency production queries without connection pooling considerations.

## Using remoteSecure() for Production

For production environments where data must be encrypted in transit, use `remoteSecure()`:

```sql
SELECT *
FROM remoteSecure(
    'prod-analytics.internal:9440',
    'warehouse',
    'fact_orders',
    'readonly_user',
    'strong_password'
)
WHERE order_date = today()
LIMIT 1000;
```

## Difference Between remote() and Distributed Engine

```text
Feature               | remote() function    | Distributed engine
----------------------+---------------------+-------------------
Setup required        | None                | CREATE TABLE needed
Query planning        | Manual              | Automatic
Reuse across queries  | Per-query           | Persistent
Suitable for          | Ad hoc, migrations  | Production workloads
TLS support           | remoteSecure()      | Config-based
```

## Summary

`remote()` and `remoteSecure()` table functions in ClickHouse enable real-time querying of data on other ClickHouse servers without defining a Distributed table. They are ideal for ad hoc cross-server analytics, data migrations with `INSERT INTO ... SELECT`, and federated queries across shards. Use `remoteSecure()` when the connection must be encrypted. For production workloads with repeated cross-server queries, consider the Distributed table engine for better performance and query planner integration.
