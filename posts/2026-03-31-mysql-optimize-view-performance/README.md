# How to Optimize View Performance in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, View, Performance, Query Optimization, Index

Description: Learn how MySQL view algorithms (MERGE vs TEMPTABLE) affect performance, when indexes apply, and how to structure views for optimal query execution.

---

## How MySQL Executes Views

MySQL uses one of two internal algorithms when executing a view:

- **MERGE** - the view definition is merged into the outer query, and the combined query is optimized as a whole. Indexes on base tables are fully usable.
- **TEMPTABLE** - the view is fully materialized into a temporary table first, and the outer query runs against that temp table. Indexes on base tables cannot be used for the outer query's WHERE clause.

MySQL chooses the algorithm automatically, but you can specify it explicitly.

## Checking Which Algorithm a View Uses

```sql
EXPLAIN SELECT * FROM my_view WHERE customer_id = 42;
```

Look at the `Extra` column. If you see "Using temporary", the view is being materialized. For MERGE views, the EXPLAIN output reflects the merged base table query.

## Specifying the ALGORITHM Explicitly

```sql
-- Force MERGE (allows index usage from base tables)
CREATE ALGORITHM = MERGE VIEW active_orders AS
SELECT o.id, o.customer_id, o.amount, o.status
FROM orders o
WHERE o.status = 'active';

-- Force TEMPTABLE (materializes before the outer query runs)
CREATE ALGORITHM = TEMPTABLE VIEW order_summary AS
SELECT customer_id, COUNT(*) AS cnt, SUM(amount) AS total
FROM orders
GROUP BY customer_id;
```

Views with GROUP BY, DISTINCT, UNION, or aggregate functions cannot use MERGE - MySQL falls back to TEMPTABLE automatically.

## When MERGE is Better

For simple filtered views without aggregation, MERGE lets the optimizer push additional predicates into the base query and use indexes:

```sql
CREATE ALGORITHM = MERGE VIEW us_customers AS
SELECT id, name, email
FROM customers
WHERE region = 'US';

-- The optimizer can use an index on (region, id) for this query
SELECT * FROM us_customers WHERE id = 500;
```

## When TEMPTABLE is Unavoidable

Aggregate views always materialize:

```sql
CREATE VIEW monthly_sales AS
SELECT DATE_FORMAT(order_date, '%Y-%m') AS month, SUM(amount) AS revenue
FROM orders
GROUP BY month;
```

To speed up TEMPTABLE views, ensure the base table query itself is well-indexed:

```sql
-- Index on order_date helps the GROUP BY scan
CREATE INDEX idx_orders_date ON orders(order_date);
```

## Avoiding Deep View Nesting

Nesting views more than two or three levels deep multiplies the number of temporary tables MySQL creates internally. Flatten deeply nested views or rewrite them as CTEs in MySQL 8:

```sql
-- CTE alternative to a nested view (better optimizer visibility)
WITH base AS (
  SELECT customer_id, SUM(amount) AS total FROM orders GROUP BY customer_id
),
high_value AS (
  SELECT * FROM base WHERE total > 1000
)
SELECT c.name, hv.total
FROM high_value hv
JOIN customers c ON c.id = hv.customer_id;
```

## Checking Index Usage for Views

```sql
EXPLAIN FORMAT=JSON
SELECT * FROM active_orders WHERE customer_id = 42\G
```

Look for `"access_type": "ref"` or `"access_type": "range"` - these indicate index use. `"access_type": "ALL"` means a full table scan.

## Summary

MySQL view performance hinges on the algorithm used. MERGE views are transparent to the optimizer and can leverage base table indexes; TEMPTABLE views materialize first and prevent the optimizer from pushing outer predicates into the base query. Always use EXPLAIN to verify index usage, index the columns most frequently filtered in view-using queries, and prefer CTEs over deeply nested views in MySQL 8.
