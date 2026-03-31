# Key SQL Syntax Differences Between ClickHouse and Standard SQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Syntax, Migration, Query Optimization

Description: An overview of the most important SQL syntax differences between ClickHouse and ANSI SQL or other major databases, with side-by-side examples.

---

## Why ClickHouse SQL Differs

ClickHouse implements a dialect of SQL optimized for analytical workloads. It supports most ANSI SQL constructs but introduces extensions and deviates in places that matter for performance at scale. Knowing the key differences prevents frustrating surprises.

## COUNT(*) vs count()

ClickHouse allows `count()` without the asterisk. Both work, but `count()` is idiomatic.

```sql
-- Standard SQL
SELECT COUNT(*) FROM events;

-- ClickHouse (preferred)
SELECT count() FROM events;
```

## Case Sensitivity of Function Names

Function names in ClickHouse are case-sensitive. `sum` works; `SUM` does not in most contexts.

```sql
-- Works
SELECT sum(amount), avg(price) FROM orders;

-- Fails
SELECT SUM(amount), AVG(price) FROM orders;
```

## Array Literals

```sql
-- ClickHouse array syntax
SELECT [1, 2, 3] AS nums;
SELECT array(1, 2, 3) AS nums;

-- Standard SQL (PostgreSQL style) - NOT valid in ClickHouse
SELECT ARRAY[1, 2, 3] AS nums;
```

## Tuple Literals

```sql
SELECT (1, 'hello', 3.14) AS t;
SELECT tuple(1, 'hello', 3.14) AS t;
```

## FINAL Modifier

ClickHouse's ReplacingMergeTree and CollapsingMergeTree engines keep duplicate rows until a background merge runs. `FINAL` forces deduplication at query time.

```sql
SELECT user_id, name FROM users FINAL WHERE is_active = 1;
```

There is no equivalent in standard SQL.

## GROUP BY Positional References

ClickHouse supports positional `GROUP BY` like MySQL.

```sql
SELECT toStartOfMonth(event_time), count()
FROM events
GROUP BY 1;
```

## SAMPLE Clause

ClickHouse has a native `SAMPLE` clause for approximate queries on large tables, controlled by sampling rate or row count.

```sql
SELECT count() FROM events SAMPLE 0.1;   -- 10% of data
SELECT count() FROM events SAMPLE 1000000; -- ~1M rows
```

## Lambda Functions

ClickHouse supports inline lambda functions for array processing.

```sql
SELECT arrayFilter(x -> x > 100, amounts) AS high_amounts
FROM orders;

SELECT arrayMap(x -> x * 1.1, prices) AS adjusted
FROM products;
```

## PREWHERE Clause

`PREWHERE` is a ClickHouse-specific optimization that applies a filter before reading all columns, reducing I/O.

```sql
SELECT user_id, amount
FROM payments
PREWHERE status = 'completed'
WHERE amount > 1000;
```

## Summary

ClickHouse SQL diverges from standard SQL in function name case sensitivity, array literal syntax, the `FINAL` modifier, `PREWHERE` optimization, native `SAMPLE` clauses, and lambda support in array functions. Most ANSI SELECT, JOIN, GROUP BY, and window function syntax works as expected. The biggest pitfall for newcomers is case-sensitive function names and forgetting `FINAL` when reading from deduplication table engines.
