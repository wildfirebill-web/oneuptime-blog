# How to Read and Interpret EXPLAIN Output in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, EXPLAIN, Query Optimization, Query Plan, Performance

Description: Learn to read ClickHouse EXPLAIN output to understand query plans, execution pipelines, and index usage for optimization.

---

The `EXPLAIN` statement in ClickHouse reveals how queries will be executed before you run them. There are three main forms: `EXPLAIN`, `EXPLAIN PIPELINE`, and `EXPLAIN PLAN`.

## EXPLAIN SYNTAX

Shows the parsed query AST:

```sql
EXPLAIN SYNTAX
SELECT u.name, count(o.id)
FROM users u
JOIN orders o ON u.id = o.user_id
GROUP BY u.name;
```

This is useful for verifying how ClickHouse rewrites implicit joins and aliases.

## EXPLAIN PLAN

Shows the logical query plan with operators:

```sql
EXPLAIN PLAN
SELECT region, sum(revenue)
FROM sales
WHERE sale_date >= '2025-01-01'
GROUP BY region;
```

Sample output:

```text
Expression ((Projection + Before ORDER BY))
  Aggregating
    Expression (Before GROUP BY)
      Filter (WHERE)
        ReadFromMergeTree (sales)
        Indexes:
          MinMax
            Keys: sale_date
            Condition: (sale_date >= '2025-01-01')
            Parts: 12/48
            Granules: 384/1536
```

Key fields to examine:
- **Parts**: how many parts are read vs total
- **Granules**: how many 8192-row blocks are scanned
- **MinMax**: indicates primary key pruning is active

## EXPLAIN PIPELINE

Shows the physical execution pipeline:

```sql
EXPLAIN PIPELINE
SELECT region, sum(revenue)
FROM sales
WHERE sale_date >= '2025-01-01'
GROUP BY region;
```

Sample output:

```text
(Expression)
ExpressionTransform
  (Aggregating)
  Resize 1 → 1
    AggregatingTransform × 4
      StrictResize 4 → 4
        (Expression)
        ExpressionTransform × 4
          (Filter)
          FilterTransform × 4
            (ReadFromMergeTree)
            MergeTreeThread × 4 0 → 1
```

The `× 4` annotations mean 4 parallel threads are used. If you see `× 1` on read steps, the query is not parallelized.

## Interpreting Poor Plans

Signs of an inefficient plan:

```text
-- Too many granules scanned (low selectivity)
Granules: 8192/8192  -- reading everything

-- No index used
Indexes: (none)

-- Only 1 thread reading
MergeTreeThread × 1
```

## Forcing Better Index Use

If index is not being used, ensure the WHERE clause matches the ORDER BY key:

```sql
-- Table ordered by (region, sale_date)
-- This WILL use the primary index:
SELECT sum(revenue) FROM sales WHERE region = 'US' AND sale_date >= '2025-01-01';

-- This will NOT use the primary index efficiently:
SELECT sum(revenue) FROM sales WHERE sale_date >= '2025-01-01';
```

## Summary

ClickHouse `EXPLAIN PLAN` shows the logical execution steps and index pruning statistics (parts and granules scanned). `EXPLAIN PIPELINE` shows the physical execution with parallelism. Look for high granule counts indicating full scans, missing index usage, and single-threaded reads as signs that a query needs optimization.
