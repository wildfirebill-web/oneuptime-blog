# How to Use MaterializedPostgreSQL Database Engine in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MaterializedPostgreSQL, PostgreSQL, CDC, Database Engine

Description: Learn how to replicate an entire PostgreSQL database into ClickHouse using the MaterializedPostgreSQL database engine and logical replication.

---

The MaterializedPostgreSQL database engine replicates an entire PostgreSQL database (or selected tables) into ClickHouse. Unlike the table-level MaterializedPostgreSQL engine, this database-level engine manages multiple tables simultaneously, each mapped to a ReplacingMergeTree table in ClickHouse. It uses PostgreSQL's logical replication slot for incremental change capture.

## PostgreSQL Setup

Enable logical replication and create a replication user:

```sql
-- postgresql.conf must have: wal_level = logical

-- Create replication slot and publication in PostgreSQL
CREATE PUBLICATION clickhouse_pub FOR ALL TABLES;
```

```sql
-- Create replication user
CREATE ROLE ch_user WITH REPLICATION LOGIN PASSWORD 'strongpassword';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO ch_user;
```

## Creating the Database in ClickHouse

```sql
CREATE DATABASE pg_replica
ENGINE = MaterializedPostgreSQL(
    '10.0.0.5:5432',
    'production',
    'ch_user',
    'strongpassword'
)
SETTINGS
    materialized_postgresql_schema = 'public',
    materialized_postgresql_tables_list = 'orders,customers,products';
```

The `materialized_postgresql_tables_list` setting lets you replicate only specific tables. Omit it to replicate all tables in the schema.

## Querying Replicated Data

```sql
USE pg_replica;

SELECT
    c.customer_id,
    c.name,
    count(o.order_id) AS total_orders,
    sum(o.amount) AS total_spent
FROM customers AS c
INNER JOIN orders AS o USING (customer_id)
WHERE o.created_at >= '2024-01-01'
GROUP BY c.customer_id, c.name
ORDER BY total_spent DESC
LIMIT 20;
```

## Monitoring Replication

```sql
-- Check database status
SELECT *
FROM system.databases
WHERE name = 'pg_replica';

-- Query the internal status table
SELECT *
FROM pg_replica._materialized_postgresql_tables_list;
```

## Handling Deleted Rows

Deleted rows in PostgreSQL are not physically removed from ClickHouse; they are marked with `_sign = -1`. Use FINAL or explicit filtering:

```sql
-- Get non-deleted rows
SELECT *
FROM pg_replica.orders FINAL
WHERE _sign = 1
  AND status = 'pending';
```

## Adding Tables Dynamically

You can add new tables to replication without recreating the database:

```sql
ALTER DATABASE pg_replica MODIFY SETTING
    materialized_postgresql_tables_list = 'orders,customers,products,invoices';
```

## Schema Change Limitations

MaterializedPostgreSQL handles some DDL:
- ADD COLUMN: Supported
- DROP TABLE: Supported
- CREATE TABLE: Supported (if included in publication)

Unsupported DDL (like RENAME COLUMN) requires manual intervention and a database recreation.

## Summary

The MaterializedPostgreSQL database engine is the recommended way to replicate multiple PostgreSQL tables into ClickHouse for analytics. It automates initial snapshots, CDC via logical replication, and schema management for a full database. Always filter on `_sign = 1` or use FINAL to exclude deleted rows, and monitor the replication status regularly to catch any lag or errors.
