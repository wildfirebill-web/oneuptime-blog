# How to Query PostgreSQL Tables Directly from ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, PostgreSQL, PostgreSQL Engine, Federation, External Table

Description: Query PostgreSQL tables directly from ClickHouse using the PostgreSQL table engine and postgresql() table function for live cross-database analytics.

---

## Overview

ClickHouse can query PostgreSQL tables in real-time using the `PostgreSQL` table engine and the `postgresql()` table function. This enables federation queries that join PostgreSQL operational data with ClickHouse analytical data without a separate ETL step.

## Ad-Hoc Query with postgresql() Function

```sql
SELECT
    user_id,
    email,
    subscription_tier,
    created_at
FROM postgresql('pg-host:5432', 'app_db', 'users', 'reader', 'password')
WHERE subscription_tier = 'pro'
  AND created_at >= '2026-01-01'
LIMIT 50;
```

## Create a Persistent PostgreSQL Table

```sql
CREATE TABLE pg_users (
    user_id           UInt64,
    email             String,
    subscription_tier LowCardinality(String),
    created_at        DateTime
)
ENGINE = PostgreSQL('pg-host:5432', 'app_db', 'users', 'reader', 'password');
```

ClickHouse queries PostgreSQL at execution time. No data is stored locally.

## Join PostgreSQL and ClickHouse Data

```sql
SELECT
    u.subscription_tier,
    count(DISTINCT e.user_id)  AS active_users,
    sum(e.revenue)             AS total_revenue,
    avg(e.revenue)             AS arpu
FROM events AS e
JOIN pg_users AS u ON e.user_id = u.user_id
WHERE e.event_ts >= today() - 30
GROUP BY u.subscription_tier;
```

This query joins a local ClickHouse `events` table with a live PostgreSQL `users` table.

## Insert PostgreSQL Data into ClickHouse

Load fresh data on a schedule:

```sql
INSERT INTO users_local
SELECT user_id, email, subscription_tier, created_at
FROM postgresql('pg-host:5432', 'app_db', 'users', 'reader', 'pass')
WHERE updated_at >= now() - INTERVAL 1 HOUR;
```

## Use a PostgreSQL Dictionary

For fast attribute lookups, use a dictionary backed by PostgreSQL:

```sql
CREATE DICTIONARY pg_user_tier (
    user_id UInt64,
    tier    String
)
PRIMARY KEY user_id
SOURCE(POSTGRESQL(
    HOST 'pg-host'
    PORT 5432
    DB 'app_db'
    TABLE 'users'
    USER 'reader'
    PASSWORD 'pass'
))
LAYOUT(HASHED())
LIFETIME(300);
```

Use in queries:

```sql
SELECT
    event_id,
    user_id,
    dictGet('pg_user_tier', 'tier', user_id) AS user_tier
FROM events
WHERE event_ts >= today()
LIMIT 1000;
```

## Handle Schema Type Differences

PostgreSQL types map to ClickHouse types automatically:

```text
INTEGER       -> Int32
BIGINT        -> Int64
TEXT          -> String
TIMESTAMPTZ   -> DateTime
NUMERIC(p,s)  -> Decimal(p, s)
BOOLEAN       -> Bool
UUID          -> UUID
```

## Check Connection Health

```sql
SELECT table, last_exception
FROM system.tables
WHERE engine = 'PostgreSQL';
```

## Summary

The `PostgreSQL` table engine and `postgresql()` function give ClickHouse live access to PostgreSQL tables. Use them for federation joins, on-demand data loads, and dictionary-based attribute lookups. For high-frequency queries, load data into local MergeTree tables to avoid hitting PostgreSQL repeatedly.
