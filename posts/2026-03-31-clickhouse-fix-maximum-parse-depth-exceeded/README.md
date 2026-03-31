# How to Fix 'Maximum parse depth exceeded' in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Parser, Error, SQL, Troubleshooting

Description: Resolve 'Maximum parse depth exceeded' errors in ClickHouse caused by deeply nested SQL queries, IN lists, or complex expressions.

---

The "Maximum parse depth exceeded" error in ClickHouse occurs when the SQL parser's recursion depth limit is hit. This typically happens with deeply nested subqueries, very long `IN (...)` lists, or programmatically generated queries with many nested conditions.

## Reproduce the Error

```sql
-- Very long IN list (can trigger the error)
SELECT count() FROM events
WHERE user_id IN (1,2,3, /* ... thousands of values ... */ 9999);

-- Deeply nested subquery
SELECT * FROM (
  SELECT * FROM (
    SELECT * FROM (/* ... 100 levels deep ... */)
  )
);
```

## Increase the Parse Depth Limit

The default maximum parse depth is configurable:

```sql
SET max_parser_depth = 2000;
SELECT ... -- your complex query
```

Set it persistently in `users.xml`:

```xml
<profiles>
  <default>
    <max_parser_depth>2000</max_parser_depth>
  </default>
</profiles>
```

## Rewrite Long IN Lists

For thousands of values in an `IN` clause, use a temporary table or a JOIN instead:

```sql
-- Create a temporary table with the values
CREATE TEMPORARY TABLE filter_ids (id UInt64) ENGINE = Memory;
INSERT INTO filter_ids VALUES (1),(2),(3) /* ... all IDs */;

-- Use a JOIN
SELECT e.*
FROM events e
INNER JOIN filter_ids f ON e.user_id = f.id;
```

Or use a `VALUES` subquery:

```sql
SELECT count()
FROM events
WHERE user_id IN (
    SELECT id FROM (
        SELECT arrayJoin([1, 2, 3, 4, 5]) AS id
    )
);
```

## Flatten Nested Subqueries

Replace deeply nested subqueries with CTEs (Common Table Expressions):

```sql
-- Instead of deeply nested subqueries:
WITH
    base AS (SELECT id, value FROM raw_data WHERE active = 1),
    aggregated AS (SELECT id, sum(value) AS total FROM base GROUP BY id),
    ranked AS (SELECT id, total, rank() OVER (ORDER BY total DESC) AS rnk FROM aggregated)
SELECT * FROM ranked WHERE rnk <= 10;
```

## Check Generator Code

If the query is generated programmatically, add a depth/length guard:

```text
- Limit IN lists to 1000 values maximum
- Use parameterized queries or JOIN-based filtering for larger sets
- Avoid recursive query generation without a depth counter
```

## Summary

"Maximum parse depth exceeded" is caused by overly complex SQL structure. Raise `max_parser_depth` as a quick fix, but the proper solution is to refactor long `IN` lists into JOINs with temporary tables, replace nested subqueries with CTEs, and add safeguards to any code that dynamically generates ClickHouse SQL.
