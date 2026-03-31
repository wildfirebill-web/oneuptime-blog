# How to Use View and MaterializedView Engines in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, View, Materialized View, Table Engine, Query Optimization

Description: Understand the View and MaterializedView table engines in ClickHouse - how they differ, when to use each, and how to create them effectively.

---

ClickHouse provides two view-based table engines: View and MaterializedView. Both let you define derived datasets, but they work very differently under the hood. Choosing the right one has significant performance implications.

## The View Table Engine

A View in ClickHouse is a stored query. It does not store any data itself. Every time you query a view, ClickHouse executes the underlying SELECT statement against the base tables.

```sql
CREATE VIEW daily_signups AS
SELECT
    toDate(created_at) AS signup_date,
    count() AS total_signups
FROM users
GROUP BY signup_date;

SELECT * FROM daily_signups
WHERE signup_date >= today() - 7;
```

Views are useful for:
- Simplifying complex queries into reusable named objects
- Providing a stable interface while the underlying schema evolves
- Controlling what columns users can query without row-level security

### Limitations of Views

Because a view reruns its SELECT on every access, it does not improve query performance. For expensive aggregations over large tables, views will be slow. They also do not support indexing or partitioning since they hold no data.

## The MaterializedView Table Engine

A MaterializedView incrementally computes and stores results as data is inserted into the source table. Rather than re-executing the query at read time, ClickHouse writes the aggregated or transformed results to a target table on every insert.

```sql
-- First create the destination table
CREATE TABLE daily_signups_mv_storage
(
    signup_date Date,
    total_signups UInt64
)
ENGINE = SummingMergeTree
ORDER BY signup_date;

-- Then create the materialized view
CREATE MATERIALIZED VIEW daily_signups_mv
TO daily_signups_mv_storage
AS
SELECT
    toDate(created_at) AS signup_date,
    count() AS total_signups
FROM users
GROUP BY signup_date;
```

Now every INSERT into `users` will trigger the materialized view to compute incremental results and append them to `daily_signups_mv_storage`. Combined with SummingMergeTree, duplicates are merged over time.

### Backfilling a Materialized View

MaterializedViews only process new inserts after they are created. To backfill historical data:

```sql
INSERT INTO daily_signups_mv_storage
SELECT
    toDate(created_at) AS signup_date,
    count() AS total_signups
FROM users
GROUP BY signup_date;
```

### Chaining Materialized Views

You can chain materialized views, where the output of one feeds another. This is useful for multi-stage aggregation pipelines, though it increases write amplification and complexity.

## View vs. MaterializedView Comparison

| Feature | View | MaterializedView |
|---|---|---|
| Stores data | No | Yes (in target table) |
| Query cost | High (reruns SQL) | Low (reads pre-computed) |
| Write overhead | None | Yes (on every insert) |
| Real-time | Yes | Near real-time (insert-driven) |
| Best for | Simple abstraction | Pre-aggregation, dashboards |

## Summary

Use the View engine when you want a reusable query abstraction without storing data. Use the MaterializedView engine when you need fast read performance on aggregated or transformed data, accepting the trade-off of additional write overhead. Most production ClickHouse deployments rely heavily on MaterializedViews to keep dashboards and reports fast without scanning raw tables.
