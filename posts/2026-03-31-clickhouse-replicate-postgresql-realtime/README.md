# How to Replicate Data from PostgreSQL to ClickHouse in Real-Time

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, PostgreSQL, Replication, CDC, Real-Time Analytics

Description: Set up real-time replication from PostgreSQL to ClickHouse using the MaterializedPostgreSQL engine or Debezium CDC for live analytical queries.

---

## Overview

ClickHouse supports two approaches for real-time PostgreSQL replication: the built-in `MaterializedPostgreSQL` database engine and a Debezium-based CDC pipeline via Kafka. Both capture PostgreSQL logical replication events and apply them to ClickHouse tables.

## Approach 1: MaterializedPostgreSQL Engine

### Configure PostgreSQL

Enable logical replication in `postgresql.conf`:

```text
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

Create a replication user:

```sql
CREATE USER ch_replicator WITH REPLICATION LOGIN PASSWORD 'strong_password';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO ch_replicator;
```

Create a publication for the tables you want to replicate:

```sql
CREATE PUBLICATION ch_publication FOR TABLE orders, customers, products;
```

### Create the Database in ClickHouse

```sql
CREATE DATABASE pg_replica
ENGINE = MaterializedPostgreSQL('pg-host:5432', 'source_db', 'ch_replicator', 'strong_password')
SETTINGS
    materialized_postgresql_replication_slot = 'ch_replication_slot',
    materialized_postgresql_tables_list = 'orders,customers,products';
```

ClickHouse creates a logical replication slot and begins streaming changes.

### Query Replicated Data

```sql
SELECT
    c.name,
    count(o.order_id)    AS num_orders,
    sum(o.total_amount)  AS lifetime_value
FROM pg_replica.orders AS o
JOIN pg_replica.customers AS c USING (customer_id)
WHERE o.status = 'shipped'
GROUP BY c.name
ORDER BY lifetime_value DESC;
```

## Approach 2: Debezium CDC via Kafka

For production workloads, Debezium provides more robust CDC:

```bash
curl -X POST http://kafka-connect:8083/connectors \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "pg-source",
    "config": {
      "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
      "database.hostname": "pg-host",
      "database.port": "5432",
      "database.user": "ch_replicator",
      "database.password": "strong_password",
      "database.dbname": "source_db",
      "database.server.name": "pg",
      "table.include.list": "public.orders",
      "plugin.name": "pgoutput"
    }
  }'
```

Consume the Debezium topic in ClickHouse:

```sql
CREATE TABLE orders_cdc_queue (
    before  String,
    after   String,
    op      String
) ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list  = 'pg.public.orders',
    kafka_group_name  = 'clickhouse-cdc',
    kafka_format      = 'JSONEachRow';
```

## Monitor Replication

```sql
SELECT *
FROM system.databases
WHERE engine = 'MaterializedPostgreSQL';
```

Check replication slot lag from PostgreSQL:

```sql
SELECT slot_name, lag_bytes
FROM pg_replication_slots
WHERE slot_name = 'ch_replication_slot';
```

## Summary

Use `MaterializedPostgreSQL` for simple, low-traffic replication needs. Use Debezium with Kafka for high-volume, production-grade CDC where you need schema evolution support, exactly-once semantics, and fine-grained control over transformation logic.
