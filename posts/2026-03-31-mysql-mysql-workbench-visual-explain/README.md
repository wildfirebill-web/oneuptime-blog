# How to Use Visual EXPLAIN in MySQL Workbench

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Workbench, Explain, Performance, Query

Description: Learn how to use MySQL Workbench's Visual EXPLAIN feature to analyze query execution plans graphically and identify performance bottlenecks.

---

## Introduction

MySQL Workbench's Visual EXPLAIN renders a query's execution plan as a flowchart, making it far easier to understand than reading raw `EXPLAIN` output. Color-coded nodes highlight expensive operations like full table scans and the diagram shows join order, index usage, estimated row counts, and access types at a glance.

## Running Visual EXPLAIN

Write your query in the SQL editor, then click the **Explain Current Statement** button (magnifying glass with lightning bolt), or use:

```text
Query > Explain Current Statement
```

Example query:

```sql
SELECT
  o.id,
  c.email,
  o.total,
  o.status
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.status = 'pending'
  AND o.created_at > '2026-01-01'
ORDER BY o.created_at DESC
LIMIT 50;
```

## Reading the Visual EXPLAIN Diagram

The diagram shows nodes for each operation in the query plan. Nodes are color-coded by cost:

```text
Blue (light)   - Low cost operation (index lookup)
Green          - Moderate cost
Orange         - Higher cost (index range scan)
Red            - Expensive operation (full table scan)
```

Each node shows:
- **Table name** - which table is being accessed
- **Access type** - `ref`, `range`, `index`, `ALL`
- **Rows** - estimated number of rows examined
- **Key** - index used (or `NULL` if none)
- **Extra** - additional info (`Using filesort`, `Using temporary`)

## Understanding Access Types

```text
system     - table has exactly one row (cheapest)
const      - one matching row found via PK or unique index
eq_ref     - one row per join key (unique index lookup)
ref        - non-unique index lookup
range      - index range scan
index      - full index scan (better than ALL)
ALL        - full table scan (most expensive, red node)
```

A red `ALL` node signals a missing index. The tooltip on each node shows:

```text
Table: orders
Access Type: ALL
Rows Examined: 1,250,000
Key: (none - no index used)
Extra: Using where; Using filesort
```

## Fixing Issues Found in Visual EXPLAIN

If the diagram shows a red full table scan on `orders.status`:

```sql
-- Add a composite index
ALTER TABLE orders ADD INDEX idx_status_created (status, created_at);
```

Re-run Visual EXPLAIN. The node turns blue with:

```text
Access Type: range
Key: idx_status_created
Rows Examined: 342
```

## Comparing EXPLAIN Formats

Workbench shows three views in tabs:

1. **Visual** - the flowchart diagram
2. **Tabular** - traditional `EXPLAIN` table output
3. **JSON** - `EXPLAIN FORMAT=JSON` with detailed cost estimates

The JSON format provides cost data not visible in tabular output:

```json
{
  "query_block": {
    "cost_info": {
      "query_cost": "148.50"
    },
    "table": {
      "table_name": "orders",
      "access_type": "range",
      "key": "idx_status_created",
      "rows_examined_per_scan": 342,
      "cost_info": {
        "read_cost": "34.20",
        "eval_cost": "3.42"
      }
    }
  }
}
```

## Using EXPLAIN ANALYZE

For actual execution statistics (MySQL 8.0.18+), use:

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'pending' LIMIT 10;
```

This shows actual rows vs. estimated rows, helping identify when the query optimizer makes bad estimates due to stale statistics. Update statistics with:

```sql
ANALYZE TABLE orders;
```

## Summary

MySQL Workbench Visual EXPLAIN makes query performance analysis accessible by visualizing execution plans as color-coded diagrams. Red nodes indicate expensive full table scans that need indexes, while blue nodes indicate efficient index lookups. Use it alongside the JSON EXPLAIN and EXPLAIN ANALYZE for a complete picture of query cost and optimizer behavior before and after adding indexes.
