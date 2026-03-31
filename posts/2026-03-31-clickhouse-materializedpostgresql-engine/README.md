# How to Use MaterializedPostgreSQL Engine in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MaterializedPostgreSQL, PostgreSQL, CDC, Replication

Description: Learn how to use the MaterializedPostgreSQL table engine to replicate a single PostgreSQL table into ClickHouse using logical replication.

---

The MaterializedPostgreSQL table engine replicates a single PostgreSQL table into ClickHouse using PostgreSQL's logical replication protocol. Changes in PostgreSQL (INSERTs, UPDATEs, DELETEs) are streamed into ClickHouse in near real-time, making it suitable for offloading analytics from transactional databases.

## Prerequisites

PostgreSQL must have logical replication enabled:

```sql
-- In postgresql.conf
-- wal_level = logical

-- On the PostgreSQL side:
SELECT pg_create_logical_replication_slot('clickhouse_slot', 'pgoutput');
```

The ClickHouse user connecting to PostgreSQL needs REPLICATION privileges:

```sql
-- In PostgreSQL
CREATE ROLE clickhouse_repl WITH REPLICATION LOGIN PASSWORD 'secret';
GRANT SELECT ON TABLE orders TO clickhouse_repl;
```

## Creating the Table

```sql
CREATE TABLE orders_replica
ENGINE = MaterializedPostgreSQL(
    'host:5432',
    'mydb',
    'orders',
    'clickhouse_repl',
    'secret'
);
```

ClickHouse will perform an initial snapshot of the `orders` table and then stream ongoing changes.

## Querying Replicated Data

```sql
SELECT
    order_id,
    customer_id,
    status,
    total_amount
FROM orders_replica
WHERE created_at >= today() - 7
ORDER BY created_at DESC
LIMIT 100;
```

## Checking Replication Status

Monitor the replication state through system tables:

```sql
SELECT
    name,
    value
FROM system.settings
WHERE name LIKE '%postgresql%';
```

For more detail, check the ClickHouse logs for messages from the MaterializedPostgreSQL background thread.

## Handling Schema Changes

MaterializedPostgreSQL does not automatically handle DDL changes in PostgreSQL. If you add a column in PostgreSQL, you need to recreate the materialized table in ClickHouse:

```sql
DETACH TABLE orders_replica;
-- Make schema changes in PostgreSQL
-- Then recreate:
DROP TABLE orders_replica;
CREATE TABLE orders_replica ENGINE = MaterializedPostgreSQL(...);
```

## Difference from PostgreSQL Table Engine

The regular PostgreSQL engine reads data live from PostgreSQL on each query. MaterializedPostgreSQL stores a local copy in ClickHouse and keeps it synchronized via CDC:

```text
PostgreSQL engine:         Live query -> higher latency, PostgreSQL load
MaterializedPostgreSQL:    Local replica -> faster reads, zero PostgreSQL query load
```

## Limitations

- Only works with tables that have a primary key in PostgreSQL.
- Does not support TRUNCATE propagation.
- For replicating multiple tables or an entire database, use the MaterializedPostgreSQL database engine instead.

## Summary

The MaterializedPostgreSQL table engine provides a simple way to create a live replica of a single PostgreSQL table inside ClickHouse. It is ideal for analytics on transactional data without burdening your PostgreSQL instance with heavy read queries. For larger-scale replication needs, consider using the MaterializedPostgreSQL database engine to replicate entire schemas.
