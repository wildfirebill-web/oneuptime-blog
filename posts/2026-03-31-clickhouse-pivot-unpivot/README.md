# How to Pivot and Unpivot Data in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Pivot, Unpivot, Analytics, Conditional Aggregation

Description: Learn how to pivot rows to columns and unpivot columns to rows in ClickHouse using conditional aggregation and arrayJoin techniques.

---

## Pivoting in ClickHouse

ClickHouse does not have a built-in `PIVOT` statement like some databases, but conditional aggregation with `sumIf`, `countIf`, and `CASE WHEN` achieves the same result. For dynamic pivots, `arrayJoin` and `groupArray` provide flexibility.

## Static Pivot with Conditional Aggregation

Convert row values into columns. Pivot daily revenue by product category:

```sql
SELECT
    toDate(event_time) AS day,
    sumIf(revenue, category = 'electronics') AS electronics,
    sumIf(revenue, category = 'clothing') AS clothing,
    sumIf(revenue, category = 'books') AS books,
    sumIf(revenue, category = 'home') AS home
FROM sales
WHERE event_time >= today() - 30
GROUP BY day
ORDER BY day;
```

## Multi-Value Pivot with CASE WHEN

For more complex pivoting logic:

```sql
SELECT
    user_id,
    sum(CASE WHEN metric = 'page_views' THEN value ELSE 0 END) AS page_views,
    sum(CASE WHEN metric = 'clicks' THEN value ELSE 0 END) AS clicks,
    sum(CASE WHEN metric = 'conversions' THEN value ELSE 0 END) AS conversions
FROM user_metrics
WHERE event_date = today() - 1
GROUP BY user_id
ORDER BY page_views DESC;
```

## Unpivot with arrayJoin

Convert columns back to rows using `arrayJoin`:

```sql
SELECT
    day,
    metric_name,
    metric_value
FROM (
    SELECT
        day,
        ['electronics', 'clothing', 'books'] AS names,
        [electronics_rev, clothing_rev, books_rev] AS values
    FROM pivoted_sales
)
ARRAY JOIN names AS metric_name, values AS metric_value
ORDER BY day, metric_name;
```

## Dynamic Pivot with groupArray

Build a pivot table where column names are not known at query time:

```sql
SELECT
    user_id,
    groupArray(metric) AS metric_names,
    groupArray(total) AS metric_values
FROM (
    SELECT user_id, metric, sum(value) AS total
    FROM user_metrics
    WHERE event_date >= today() - 7
    GROUP BY user_id, metric
)
GROUP BY user_id;
```

Process the resulting arrays in application code to render the pivot table.

## Transposing a Result Set

Transpose 7 days of metrics into a row-per-metric format:

```sql
WITH daily AS (
    SELECT
        toDate(event_time) AS day,
        count() AS events,
        uniq(user_id) AS users
    FROM user_events
    WHERE event_time >= today() - 7
    GROUP BY day
)
SELECT 'events' AS metric, groupArray(events) AS values, groupArray(day) AS days
FROM daily
UNION ALL
SELECT 'users', groupArray(users), groupArray(day)
FROM daily;
```

## Summary

ClickHouse pivots data using conditional aggregation (`sumIf`, `CASE WHEN`) for static pivots with known column values, and `arrayJoin` for unpivoting arrays back to rows. Dynamic pivots return arrays that application code renders into tabular form. These patterns handle virtually any data reshaping need without a dedicated `PIVOT` keyword.
