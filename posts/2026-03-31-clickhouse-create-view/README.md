# How to Create a View in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, DDL, View, SELECT

Description: Learn how to create regular views in ClickHouse using CREATE VIEW, including IF NOT EXISTS, OR REPLACE, and how views compare to materialized views.

---

Views in ClickHouse are named SQL queries stored in the catalog. Every time you query a view, ClickHouse re-executes the underlying SELECT against the current state of the base tables. Views simplify complex queries, encapsulate business logic, and make schemas easier to evolve without breaking downstream consumers. If you need pre-computed, incrementally maintained results instead, you want a materialized view - but regular views are a good starting point because they require no extra storage and always return fresh data.

## Basic CREATE VIEW Syntax

```sql
CREATE VIEW [IF NOT EXISTS] [db.]view_name AS
SELECT ...;
```

A minimal example that abstracts a filtered projection:

```sql
CREATE VIEW active_users AS
SELECT
    user_id,
    email,
    created_at
FROM users
WHERE status = 'active';
```

After creation, the view is queryable exactly like a table:

```sql
SELECT * FROM active_users WHERE created_at > now() - INTERVAL 7 DAY;
```

ClickHouse rewrites this query by inlining the view definition at execution time.

## IF NOT EXISTS

Adding `IF NOT EXISTS` prevents an error if the view already exists. This is essential in migration scripts that may run multiple times:

```sql
CREATE VIEW IF NOT EXISTS active_users AS
SELECT user_id, email, created_at
FROM users
WHERE status = 'active';
```

## OR REPLACE

`OR REPLACE` atomically drops and recreates the view in one statement. Use this when you need to update a view's definition:

```sql
CREATE OR REPLACE VIEW active_users AS
SELECT
    user_id,
    email,
    created_at,
    last_login   -- newly added column
FROM users
WHERE status = 'active';
```

`OR REPLACE` is safer than a manual `DROP VIEW` followed by `CREATE VIEW` because there is no window where the view is absent.

## Querying Through Views

Views are fully composable. You can join views, filter them, aggregate over them, and use them as subqueries:

```sql
-- Aggregation over a view
SELECT
    toStartOfDay(created_at) AS signup_date,
    count() AS new_users
FROM active_users
GROUP BY signup_date
ORDER BY signup_date;

-- Join two views
SELECT
    u.user_id,
    u.email,
    o.total_orders
FROM active_users u
INNER JOIN (
    SELECT user_id, count() AS total_orders
    FROM orders
    GROUP BY user_id
) o ON u.user_id = o.user_id;
```

## Parameterized Views

ClickHouse supports parameterized views that accept arguments at query time, similar to stored procedures:

```sql
CREATE VIEW events_by_type AS
SELECT
    event_time,
    user_id,
    properties
FROM events
WHERE event_type = {event_type: String}
  AND event_date >= {start_date: Date}
  AND event_date <= {end_date: Date};
```

Call the view with named parameters:

```sql
SELECT *
FROM events_by_type(
    event_type = 'purchase',
    start_date = '2025-01-01',
    end_date   = '2025-01-31'
);
```

Parameterized views avoid the need to create many near-identical views for slight variations of a query.

## Inspecting Views

List all views in a database:

```sql
SHOW TABLES FROM my_database;

-- Filter to only views
SELECT name, engine
FROM system.tables
WHERE database = 'my_database'
  AND engine = 'View';
```

Show the underlying definition:

```sql
SHOW CREATE VIEW active_users;
```

## Dropping a View

```sql
DROP VIEW IF EXISTS active_users;
```

## Views vs. Materialized Views

| Aspect | Regular View | Materialized View |
|---|---|---|
| Storage | None | Separate table |
| Data freshness | Always current | Incremental (trigger-based) |
| Query performance | Re-executes SELECT | Reads pre-computed data |
| Use case | Encapsulate logic | Pre-aggregate large tables |

Regular views are free in terms of storage. The trade-off is that every query against a view pays the full scan cost of the underlying tables. For dashboards that aggregate billions of rows, a materialized view backed by `AggregatingMergeTree` or `SummingMergeTree` will be far faster.

## Practical Example - Layered Analytics Schema

A common pattern is to layer views on top of raw tables to create a clean semantic layer:

```sql
-- Raw events table (append-only)
CREATE TABLE raw_events
(
    event_time  DateTime,
    user_id     UInt64,
    event_type  LowCardinality(String),
    revenue     Nullable(Decimal(18,2))
)
ENGINE = MergeTree()
ORDER BY (event_type, user_id, event_time);

-- Clean view with type coercion and nulls handled
CREATE VIEW events AS
SELECT
    event_time,
    user_id,
    event_type,
    coalesce(revenue, 0) AS revenue
FROM raw_events;

-- Business-level view for revenue analysis
CREATE VIEW revenue_events AS
SELECT *
FROM events
WHERE revenue > 0;
```

Downstream users query `revenue_events` without knowing about the raw schema details.

## Summary

Regular views in ClickHouse are lightweight query aliases that execute their SELECT on every access, always returning fresh data without consuming any extra storage. Use `IF NOT EXISTS` for idempotent migrations, `OR REPLACE` for in-place updates, and parameterized views when you want query-time flexibility. When query latency over large datasets matters more than storage cost, graduate to materialized views instead.
