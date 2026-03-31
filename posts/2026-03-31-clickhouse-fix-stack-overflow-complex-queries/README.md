# How to Fix "Stack overflow" in Complex ClickHouse Queries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Stack Overflow, Error, Query, Troubleshooting

Description: Diagnose and fix stack overflow errors in ClickHouse caused by deeply recursive SQL expressions, nested functions, or query analyzer depth.

---

A stack overflow in ClickHouse occurs when the server's query execution or analysis phase exceeds the call stack size. This is most common with very complex nested SQL expressions, deeply recursive CTEs, or certain analyzer traversals on large query trees.

## Recognize the Error

The error message typically looks like:

```text
Code: 2002. DB::Exception: Stack overflow, max size: 65536
```

or

```text
Received exception from server (version 24.x):
Code: 2002. DB::Exception: Stack size too large.
```

## Increase Stack Size for ClickHouse Threads

In `config.xml`, increase the thread stack size:

```xml
<thread_stack_size>8388608</thread_stack_size>
```

The default is typically 4 MB. Setting 8 MB or 16 MB gives more headroom for complex queries.

Restart the server after this change:

```bash
sudo systemctl restart clickhouse-server
```

## Increase Query Analyzer Depth

For large query trees hitting analyzer limits:

```sql
SET max_analyzer_depth = 1000;
```

## Simplify Deeply Nested Expressions

Replace deeply nested function calls with intermediate columns:

```sql
-- Problematic: 50-level nesting
SELECT f1(f2(f3(f4( /* ... */ )))) FROM t;

-- Better: use intermediate expressions
WITH
    step1 AS (SELECT f4(raw) AS v1 FROM t),
    step2 AS (SELECT f3(v1) AS v2 FROM step1),
    step3 AS (SELECT f2(v2) AS v3 FROM step2)
SELECT f1(v3) FROM step3;
```

## Avoid Recursive CTEs

ClickHouse does not support recursive CTEs (as of ClickHouse 24.x). If a query has been migrated from PostgreSQL with recursive CTEs, rewrite using iterative logic or arrays:

```sql
-- Use arrayJoin for flattening hierarchies instead of recursive CTE
SELECT
    user_id,
    arrayJoin(path_array) AS step
FROM funnel_data;
```

## Limit Lambda Complexity

Deeply nested `arrayMap`, `arrayFilter`, and `arrayReduce` combinations can cause stack overflows:

```sql
-- Instead of chaining many array functions, use temporary arrays:
WITH
    filtered AS (SELECT arrayFilter(x -> x > 0, values) AS arr FROM t),
    mapped AS (SELECT arrayMap(x -> x * 2, arr) AS arr2 FROM filtered)
SELECT arraySum(arr2) FROM mapped;
```

## Summary

ClickHouse stack overflow errors stem from excessive call stack depth during query planning or execution. Increase `thread_stack_size` in `config.xml` for immediate relief, then refactor the query to reduce nesting using CTEs, intermediate expressions, or array operations. Avoid deeply chained lambda functions and simplify complex expression trees.
