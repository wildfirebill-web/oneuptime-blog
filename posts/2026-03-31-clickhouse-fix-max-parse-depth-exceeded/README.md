# How to Fix 'Maximum parse depth exceeded' in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Troubleshooting, Parser, Query, Configuration

Description: Fix the 'Maximum parse depth exceeded' error in ClickHouse caused by deeply nested SQL expressions, large IN lists, or recursive CTEs.

---

The "Maximum parse depth exceeded" error occurs when ClickHouse's SQL parser encounters a query that exceeds the default recursion depth limit. This is common with deeply nested expressions, huge IN lists, or complex CTEs.

## Understanding the Error

```text
Code: 62. DB::Exception: Syntax error: Maximum parse depth 1000 exceeded.
```

The default `max_parser_depth` is 1000. Each nesting level in SQL (subqueries, nested functions, IN lists) consumes parse depth.

## Common Causes

### Large IN Lists

```sql
-- This can easily exceed parse depth with thousands of values
SELECT * FROM events
WHERE user_id IN (1, 2, 3, 4, ... 10000 values ...);
```

### Deeply Nested CASE Expressions

```sql
-- Each CASE WHEN nesting consumes depth
SELECT CASE WHEN x = 1 THEN CASE WHEN y = 1 THEN ... END ... END;
```

### Complex Recursive CTEs

```sql
WITH RECURSIVE
    a AS (SELECT ...),
    b AS (SELECT ... FROM a),
    c AS (SELECT ... FROM b),
    ... deeply nested CTEs ...
```

## Fix 1: Increase max_parser_depth

Increase the limit for your session:

```sql
SET max_parser_depth = 5000;
SELECT * FROM events WHERE user_id IN (1, 2, 3, ...);
```

Or set it globally in the user profile:

```xml
<clickhouse>
  <profiles>
    <default>
      <max_parser_depth>5000</max_parser_depth>
    </default>
  </profiles>
</clickhouse>
```

## Fix 2: Replace IN Lists with a Table

Instead of a large IN list, insert the values into a temporary table:

```sql
-- Create a temp table with the values
CREATE TEMPORARY TABLE filter_ids (id UInt64);
INSERT INTO filter_ids VALUES (1), (2), (3), ... (10000);

-- Use a JOIN instead of IN
SELECT e.*
FROM events e
INNER JOIN filter_ids f ON e.user_id = f.id;
```

Or use the `values()` table function:

```sql
SELECT * FROM events
WHERE user_id IN (
    SELECT id FROM VALUES('id UInt64', 1, 2, 3, ... 10000)
);
```

## Fix 3: Replace IN List with a File

For very large lists, load from a file:

```bash
# Create a file with one ID per line
cat user_ids.txt
1
2
3
```

```sql
SELECT * FROM events
WHERE user_id IN (
    SELECT toUInt64(trim(line)) FROM file('/tmp/user_ids.txt', LineAsString)
);
```

## Fix 4: Use arrayJoin or array() for Flat Lists

```sql
-- Simpler parsing for small lists
SELECT * FROM events
WHERE has([1, 2, 3, 4, 5], user_id);
```

The `has()` function with an array literal may parse more efficiently than a large IN list.

## Fix 5: Flatten Nested CASE Expressions

Refactor deeply nested CASE into a multiIf():

```sql
-- Instead of nested CASE WHEN
SELECT multiIf(
    x = 1, 'one',
    x = 2, 'two',
    x = 3, 'three',
    'other'
) FROM my_table;
```

`multiIf()` is a flat function that avoids nesting depth.

## Summary

"Maximum parse depth exceeded" is fixed by increasing `max_parser_depth` for complex legitimate queries, or by refactoring large IN lists into JOIN operations with temporary tables, flattening nested CASE expressions into `multiIf()`, and breaking deeply nested CTEs into separate steps. The table-based approach is preferred for very large filter lists as it also improves query performance.
