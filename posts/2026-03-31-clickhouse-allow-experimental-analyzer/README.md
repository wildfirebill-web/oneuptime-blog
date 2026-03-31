# How to Use allow_experimental_analyzer in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Database, Query Optimization, Configuration, SQL

Description: Learn how to enable the new ClickHouse query analyzer with allow_experimental_analyzer for improved query planning, better subquery support, and more standard SQL compatibility.

---

ClickHouse shipped a completely rewritten query analysis and planning pipeline starting in version 22.6, controlled by the `allow_experimental_analyzer` setting (made the default in version 23.9). The new analyzer provides better SQL standard compliance, improved subquery handling, more aggressive query optimizations, and more accurate error messages. If you are on an older version or need to opt in explicitly, this guide explains what the analyzer changes and how to enable it safely.

## What the New Analyzer Changes

The original ClickHouse analyzer was built before modern SQL standards were fully considered. The new analyzer rewrites the entire query analysis phase from scratch and provides:

- Correct resolution of column aliases across subqueries.
- Proper `QUALIFY` clause support.
- Better handling of correlated subqueries.
- More standard `JOIN` behavior with column name resolution.
- Improved `WITH` (CTE) support including recursive CTEs.
- Better constant folding and predicate pushdown.
- More accurate error messages pointing to the actual problem.

## Checking the Current State

```sql
-- Check whether the new analyzer is active
SELECT value, changed
FROM system.settings
WHERE name = 'allow_experimental_analyzer';

-- Check the ClickHouse version
SELECT version();
```

In ClickHouse 23.9 and later, `allow_experimental_analyzer = 1` is the default. On older versions, it defaults to 0.

## Enabling the New Analyzer

Per query:

```sql
SELECT
    user_id,
    count() AS events,
    sum(revenue) AS total_revenue
FROM events
WHERE event_date >= today() - 30
GROUP BY user_id
HAVING total_revenue > 100
ORDER BY total_revenue DESC
LIMIT 100
SETTINGS allow_experimental_analyzer = 1;
```

For a session:

```sql
SET allow_experimental_analyzer = 1;

-- All subsequent queries in this session use the new analyzer
SELECT count() FROM events WHERE event_date = today();
```

In a user profile:

```xml
<clickhouse>
    <profiles>
        <default>
            <allow_experimental_analyzer>1</allow_experimental_analyzer>
        </default>
    </profiles>
</clickhouse>
```

## SQL Features That Work Better with the New Analyzer

### Correct Alias Resolution in Subqueries

```sql
-- The new analyzer correctly resolves 'event_count' alias in HAVING
SELECT
    user_id,
    count() AS event_count,
    sum(revenue) AS total
FROM events
GROUP BY user_id
HAVING event_count > 10  -- Old analyzer may not resolve this alias
SETTINGS allow_experimental_analyzer = 1;
```

### CTEs (WITH Clauses)

```sql
-- Multi-level CTEs with references between them
WITH
    active_users AS (
        SELECT DISTINCT user_id
        FROM events
        WHERE event_date >= today() - 7
    ),
    user_revenue AS (
        SELECT
            user_id,
            sum(revenue) AS total_revenue
        FROM orders
        WHERE user_id IN (SELECT user_id FROM active_users)
        GROUP BY user_id
    )
SELECT
    a.user_id,
    r.total_revenue
FROM active_users AS a
LEFT JOIN user_revenue AS r ON a.user_id = r.user_id
ORDER BY r.total_revenue DESC NULLS LAST
SETTINGS allow_experimental_analyzer = 1;
```

### QUALIFY Clause

`QUALIFY` filters results after window function computation, similar to how `HAVING` filters after `GROUP BY`:

```sql
-- Return only the top 3 events per user by revenue
SELECT
    user_id,
    event_date,
    revenue,
    rank() OVER (PARTITION BY user_id ORDER BY revenue DESC) AS rank
FROM orders
QUALIFY rank <= 3
SETTINGS allow_experimental_analyzer = 1;
```

### Improved Window Functions

```sql
-- Running total with proper frame semantics
SELECT
    event_date,
    daily_revenue,
    sum(daily_revenue) OVER (
        ORDER BY event_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_revenue
FROM (
    SELECT
        event_date,
        sum(revenue) AS daily_revenue
    FROM orders
    GROUP BY event_date
)
ORDER BY event_date
SETTINGS allow_experimental_analyzer = 1;
```

### JOIN Column Resolution

```sql
-- The new analyzer correctly resolves ambiguous column names in JOINs
SELECT
    e.user_id,
    e.event_type,
    u.country,
    u.plan_type
FROM events AS e
INNER JOIN users AS u USING (user_id)
WHERE e.event_date = today()
  AND u.country = 'US'
SETTINGS allow_experimental_analyzer = 1;
```

## Using EXPLAIN with the New Analyzer

The new analyzer produces more detailed `EXPLAIN` output:

```sql
-- Full query tree from the new analyzer
EXPLAIN QUERY TREE
SELECT user_id, count()
FROM events
WHERE event_date = today()
GROUP BY user_id
SETTINGS allow_experimental_analyzer = 1;
```

```sql
-- Execution plan
EXPLAIN PLAN
SELECT user_id, count()
FROM events
WHERE event_date = today()
GROUP BY user_id
SETTINGS allow_experimental_analyzer = 1;
```

```sql
-- Pipeline stages
EXPLAIN PIPELINE
SELECT user_id, count()
FROM events
WHERE event_date = today()
GROUP BY user_id
SETTINGS allow_experimental_analyzer = 1;
```

## Migration Considerations

If you are enabling the new analyzer on an existing deployment for the first time:

**Test your most complex queries first.** Queries that relied on undocumented or non-standard behavior of the old analyzer may return different results or raise different errors.

**Watch for alias resolution changes.** The new analyzer is stricter about forward references and alias scoping.

**Review custom dictionary and materialized view queries.** These run at write time and should be tested with the new analyzer before enabling it globally.

```sql
-- Compare output between old and new analyzer
-- Old analyzer
SELECT user_id, count() AS cnt
FROM events
GROUP BY user_id
ORDER BY cnt DESC
LIMIT 10
SETTINGS allow_experimental_analyzer = 0;

-- New analyzer
SELECT user_id, count() AS cnt
FROM events
GROUP BY user_id
ORDER BY cnt DESC
LIMIT 10
SETTINGS allow_experimental_analyzer = 1;
```

## Disabling the New Analyzer for a Specific Query

If a query behaves unexpectedly with the new analyzer and you need a quick workaround:

```sql
-- Temporarily fall back to the old analyzer for this query
SELECT *
FROM legacy_view
SETTINGS allow_experimental_analyzer = 0;
```

## Performance Impact

The new analyzer generally produces better or equal query plans. Specific improvements include:

- More aggressive constant folding reduces the number of expressions evaluated at runtime.
- Better predicate pushdown reduces the number of rows read.
- Improved subquery decorrelation converts correlated subqueries into joins.

```sql
-- Check for plan improvements
EXPLAIN PIPELINE
SELECT
    user_id,
    (SELECT count() FROM events e2 WHERE e2.user_id = e1.user_id) AS event_count
FROM users AS e1
WHERE country = 'US'
SETTINGS allow_experimental_analyzer = 1;
```

## Conclusion

The new query analyzer (`allow_experimental_analyzer = 1`) is the default in ClickHouse 23.9+ and is the recommended path for all deployments. It delivers better SQL compatibility, improved query optimization, and more accurate error messages. On older versions, enable it per-profile in `users.xml`, test your most complex queries with `EXPLAIN QUERY TREE`, and watch for alias resolution or subquery behavior differences before rolling it out globally.
