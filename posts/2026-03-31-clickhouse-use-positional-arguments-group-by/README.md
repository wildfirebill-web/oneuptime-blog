# How to Use Positional Arguments in ClickHouse GROUP BY

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, GROUP BY, Positional Arguments, SQL, Query Writing

Description: Learn how to use positional column references in ClickHouse GROUP BY and ORDER BY clauses to write shorter, more maintainable analytical queries.

---

ClickHouse supports positional arguments in GROUP BY and ORDER BY clauses, allowing you to reference SELECT columns by their position number rather than repeating the full expression. This is especially convenient for complex expressions.

## Basic Positional GROUP BY

Instead of repeating an expression:

```sql
-- Verbose: repeat the expression
SELECT
    toStartOfHour(timestamp),
    event_type,
    count()
FROM events
GROUP BY toStartOfHour(timestamp), event_type;

-- Concise: use positional references
SELECT
    toStartOfHour(timestamp),
    event_type,
    count()
FROM events
GROUP BY 1, 2;
```

Both queries are equivalent.

## Positional ORDER BY

Same syntax works for ORDER BY:

```sql
SELECT
    service,
    count() AS requests,
    avg(duration_ms) AS avg_latency
FROM http_requests
WHERE timestamp >= now() - INTERVAL 1 HOUR
GROUP BY 1
ORDER BY 2 DESC, 3 ASC;
```

Position 2 is `requests`, position 3 is `avg_latency`.

## Mixed Positional and Named References

You can mix styles:

```sql
SELECT
    toDate(timestamp) AS day,
    lower(event_type) AS event_type,
    count() AS cnt
FROM events
GROUP BY 1, event_type
ORDER BY day, cnt DESC;
```

## Enabling Positional Arguments

Positional GROUP BY is enabled by default in newer ClickHouse versions. If not, enable it:

```sql
SET enable_positional_arguments = 1;
```

## Complex Expressions Made Cleaner

Positional references shine when expressions are complex:

```sql
SELECT
    multiIf(
        duration_ms < 100, 'fast',
        duration_ms < 500, 'normal',
        duration_ms < 2000, 'slow',
        'very slow'
    ) AS latency_bucket,
    count() AS requests,
    avg(duration_ms) AS avg_ms
FROM http_requests
GROUP BY 1
ORDER BY avg_ms ASC;
```

Without positional syntax, you would need to repeat the full `multiIf(...)` expression in GROUP BY.

## ROLLUP and CUBE with Positional Arguments

Works with GROUP BY modifiers too:

```sql
SELECT
    toDate(timestamp) AS day,
    service,
    count() AS requests
FROM http_requests
GROUP BY 1, 2 WITH ROLLUP
ORDER BY day NULLS LAST, service NULLS LAST;
```

## HAVING with Positional Arguments

HAVING filters on aggregated results - you can use column aliases but not positions:

```sql
SELECT
    service,
    count() AS total,
    countIf(status_code >= 500) AS errors
FROM http_requests
GROUP BY 1
HAVING errors > 100
ORDER BY errors DESC;
```

## Summary

Positional arguments in ClickHouse GROUP BY and ORDER BY eliminate the need to repeat complex expressions and make queries more concise. They are especially useful when grouping by computed columns like `toStartOfHour(timestamp)` or `multiIf(...)` expressions. Enable them with `SET enable_positional_arguments = 1` if needed on older versions.
