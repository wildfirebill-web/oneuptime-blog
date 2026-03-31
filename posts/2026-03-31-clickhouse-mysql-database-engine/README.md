# How to Use MySQL Database Engine in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MySQL Database Engine, MySQL, Federated Query, External Database

Description: Learn how to use the MySQL database engine in ClickHouse to query MySQL tables live without copying data into ClickHouse.

---

The MySQL database engine in ClickHouse creates a virtual database that proxies queries to a live MySQL server. Tables in the specified MySQL database become queryable as ClickHouse tables. This is useful for joining MySQL operational data with ClickHouse analytics, data migration, and one-off analytical queries on MySQL tables.

## Creating the Database

```sql
CREATE DATABASE mysql_live
ENGINE = MySQL(
    '10.0.0.10:3306',
    'production_db',
    'readonly_user',
    'password'
);
```

All tables in `production_db` become accessible through `mysql_live` in ClickHouse.

## Querying MySQL Tables

```sql
SELECT
    user_id,
    email,
    created_at,
    subscription_status
FROM mysql_live.users
WHERE created_at >= '2024-01-01'
ORDER BY created_at DESC
LIMIT 50;
```

## Joining MySQL and ClickHouse Data

The most powerful use case is enriching ClickHouse analytics with MySQL master data:

```sql
SELECT
    e.ts,
    e.event_type,
    u.email,
    u.plan
FROM clickhouse_events AS e
INNER JOIN mysql_live.users AS u USING (user_id)
WHERE e.ts >= today() - 7
  AND u.plan = 'enterprise'
ORDER BY e.ts DESC;
```

ClickHouse fetches the MySQL data and performs the join in its engine.

## Listing Available Tables

```sql
SHOW TABLES FROM mysql_live;
```

## Writing Data Back to MySQL

Unlike PostgreSQL database engine, the MySQL database engine supports INSERT:

```sql
INSERT INTO mysql_live.audit_log VALUES
(now(), 'clickhouse', 'bulk_export_completed', 'rows=1000000');
```

This sends an INSERT directly to MySQL. Use with caution in production.

## Predicate Pushdown

WHERE clauses are pushed to MySQL where compatible:

```sql
-- This filter runs on MySQL, reducing data transferred to ClickHouse
SELECT * FROM mysql_live.products
WHERE category_id = 5
  AND in_stock = 1;
```

Non-pushable expressions result in a full table scan fetched to ClickHouse for local filtering.

## Performance and Limitations

```text
- Every query executes on MySQL in real time
- No caching - results are not stored in ClickHouse
- Performance is bounded by MySQL response time
- Not suitable for high-frequency heavy analytics
- For analytics, prefer MaterializedMySQL
```

## Handling MySQL Type Mappings

ClickHouse maps MySQL types automatically:

```text
MySQL INT         -> Int32
MySQL BIGINT      -> Int64
MySQL VARCHAR     -> String
MySQL DATETIME    -> DateTime
MySQL DECIMAL     -> Decimal
MySQL TINYINT(1)  -> UInt8
```

Complex types like JSON may not map cleanly and require explicit casting.

## Summary

The MySQL database engine provides a lightweight way to run federated queries against MySQL from ClickHouse. It is best suited for occasional lookups, data exploration, and migration workflows. For production analytics on MySQL data with low latency and high throughput, use the MaterializedMySQL database engine to maintain a local ClickHouse replica instead.
