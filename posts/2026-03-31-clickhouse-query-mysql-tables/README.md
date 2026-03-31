# How to Query MySQL Tables Directly from ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MySQL, MySQL Engine, Federation, External Table

Description: Query MySQL tables directly from ClickHouse using the MySQL table engine and mysql() table function without replicating data.

---

## Overview

ClickHouse provides two ways to query MySQL tables without copying data: the `MySQL` table engine for persistent access and the `mysql()` table function for ad-hoc queries. Both execute the query on the MySQL side (pushdown) and return results to ClickHouse.

## Ad-Hoc Query with mysql() Function

Use the `mysql()` table function for one-off queries:

```sql
SELECT
    order_id,
    customer_id,
    total_amount,
    status
FROM mysql('mysql-host:3306', 'shop', 'orders', 'reader', 'password')
WHERE status = 'pending'
  AND created_at >= '2026-03-01'
LIMIT 100;
```

The query condition is pushed down to MySQL when possible.

## Create a Persistent MySQL Table

For repeated access, create a MySQL engine table in ClickHouse:

```sql
CREATE TABLE mysql_orders
(
    order_id    UInt64,
    customer_id UInt64,
    amount      Decimal(18, 2),
    status      String,
    created_at  DateTime
)
ENGINE = MySQL('mysql-host:3306', 'shop', 'orders', 'reader', 'password');
```

Query it like a local table:

```sql
SELECT count(), sum(amount)
FROM mysql_orders
WHERE status = 'completed';
```

## Join MySQL and ClickHouse Tables

Combine live MySQL data with local ClickHouse analytics tables:

```sql
SELECT
    m.customer_id,
    m.email,
    a.total_orders,
    a.lifetime_value
FROM mysql('mysql-host:3306', 'shop', 'customers', 'reader', 'pass') AS m
JOIN (
    SELECT
        customer_id,
        count()      AS total_orders,
        sum(amount)  AS lifetime_value
    FROM local_orders
    GROUP BY customer_id
) AS a ON m.customer_id = a.customer_id
ORDER BY a.lifetime_value DESC
LIMIT 20;
```

## Insert from MySQL into ClickHouse

Use the mysql function as a source for loading data:

```sql
INSERT INTO local_orders
SELECT order_id, customer_id, amount, status, created_at
FROM mysql('mysql-host:3306', 'shop', 'orders', 'reader', 'pass')
WHERE created_at >= today() - 1;
```

## Use a MySQL Dictionary for Lookups

For fast key-value lookups from MySQL, use a dictionary:

```sql
CREATE DICTIONARY customer_info (
    customer_id UInt64,
    email       String,
    tier        String
)
PRIMARY KEY customer_id
SOURCE(MYSQL(
    HOST 'mysql-host'
    PORT 3306
    DB 'shop'
    TABLE 'customers'
    USER 'reader'
    PASSWORD 'pass'
))
LAYOUT(FLAT())
LIFETIME(600);
```

Then use it in queries:

```sql
SELECT
    order_id,
    dictGet('customer_info', 'email', customer_id) AS email,
    dictGet('customer_info', 'tier', customer_id)  AS tier
FROM local_orders
WHERE order_ts >= today();
```

## Check Connection Status

```sql
SELECT * FROM system.dictionaries
WHERE source = 'MySQL';
```

## Summary

The `MySQL` table engine and `mysql()` function let you query MySQL data directly from ClickHouse for federation queries and ad-hoc analytics. For better performance on large datasets, load frequently accessed data into local MergeTree tables or use a dictionary for fast key lookups.
