# How to Configure PostgreSQL Logical Replication with IPv4 Publishers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: PostgreSQL, Logical Replication, IPv4, Database, Publication, Subscription, Migration

Description: Learn how to configure PostgreSQL logical replication to replicate specific tables from an IPv4 publisher to a subscriber for migrations and selective replication.

---

Logical replication replicates individual tables (not the entire instance) at the SQL level. Unlike streaming replication, the subscriber can be a different PostgreSQL major version, making it ideal for zero-downtime major version upgrades and cross-datacenter replication of specific tables.

## Publisher (Source) Configuration

### postgresql.conf

```ini
# /etc/postgresql/15/main/postgresql.conf (on the publisher)

listen_addresses = '10.0.0.10,127.0.0.1'

# Logical replication requires logical WAL level

wal_level = logical

max_replication_slots = 10
max_wal_senders = 10
```

### pg_hba.conf

```text
# /etc/postgresql/15/main/pg_hba.conf
# Allow the subscriber to connect for replication
host  mydb  replicator  10.0.0.11/32  scram-sha-256
```

### Create the Replication User and Publication

```sql
-- Create a user with replication privileges
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'ReplPass123!';

-- Grant SELECT on the tables to be replicated
GRANT SELECT ON TABLE orders, customers, products TO replicator;

-- Create a publication for specific tables
CREATE PUBLICATION my_publication FOR TABLE orders, customers, products;

-- Or publish all tables in a schema:
-- CREATE PUBLICATION my_pub FOR ALL TABLES;
```

## Subscriber (Destination) Configuration

### Prepare the Tables on the Subscriber

The tables must exist on the subscriber before creating the subscription.

```sql
-- On the subscriber: create matching table structures
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT,
    total NUMERIC,
    created_at TIMESTAMPTZ
);

CREATE TABLE customers (
    id BIGINT PRIMARY KEY,
    name TEXT,
    email TEXT
);

-- Grant INSERT/UPDATE/DELETE to the replication apply worker
ALTER TABLE orders OWNER TO replicator;
ALTER TABLE customers OWNER TO replicator;
```

### Create the Subscription

```sql
-- On the subscriber: connect to the publisher and subscribe
CREATE SUBSCRIPTION my_subscription
CONNECTION 'host=10.0.0.10 port=5432 dbname=mydb user=replicator password=ReplPass123!'
PUBLICATION my_publication;
```

PostgreSQL will immediately copy existing data and then stream changes.

## Monitoring Replication

```sql
-- On the publisher: check replication slots
SELECT slot_name, active, restart_lsn FROM pg_replication_slots;

-- On the subscriber: check subscription status
SELECT subname, subenabled, subconninfo FROM pg_subscription;

-- Check replication lag on the subscriber
SELECT now() - pg_last_xact_replay_timestamp() AS lag;

-- Detailed stats
SELECT * FROM pg_stat_subscription;
```

## Managing Publications and Subscriptions

```sql
-- Add a new table to an existing publication
ALTER PUBLICATION my_publication ADD TABLE invoices;

-- On the subscriber: refresh to pick up the new table
ALTER SUBSCRIPTION my_subscription REFRESH PUBLICATION;

-- Pause replication
ALTER SUBSCRIPTION my_subscription DISABLE;

-- Resume
ALTER SUBSCRIPTION my_subscription ENABLE;

-- Drop subscription
DROP SUBSCRIPTION my_subscription;
```

## Key Takeaways

- `wal_level = logical` is required on the publisher; it's a superset of `replica`.
- Tables must exist on the subscriber before creating the subscription.
- Logical replication works between different PostgreSQL major versions (e.g., PG 14 → PG 15).
- Use `ALTER SUBSCRIPTION ... REFRESH PUBLICATION` after adding tables to an existing publication.
