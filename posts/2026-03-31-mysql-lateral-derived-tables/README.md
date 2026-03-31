# How to Use LATERAL Derived Tables in MySQL 8

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, Derived Table, Lateral Join, Database, Query, MySQL 8

Description: Learn how LATERAL derived tables in MySQL 8 allow subqueries in the FROM clause to reference columns from preceding tables, enabling per-row subqueries and top-N-per-group queries.

---

A `LATERAL` derived table is a subquery in the `FROM` clause that can reference columns from tables listed earlier in the same `FROM` clause. Standard (non-lateral) derived tables are isolated: they cannot see outer query columns. `LATERAL` removes that restriction and was introduced in MySQL 8.0.14.

## Syntax

```sql
SELECT ...
FROM table1 t1
JOIN LATERAL (
    SELECT ...
    FROM table2 t2
    WHERE t2.fk = t1.pk    -- references t1, the preceding table
) AS alias ON TRUE;
```

The `ON TRUE` (or `ON 1=1`) is used when the lateral subquery already handles the filtering internally.

## Problem that LATERAL solves

Before `LATERAL`, getting the top-N rows per group required a window function or a correlated subquery. Correlated subqueries in `FROM` were not allowed to reference outer columns.

```sql
-- This fails in standard SQL: cannot reference c.customer_id in FROM subquery
SELECT c.name, recent.order_date
FROM customers c,
     (SELECT order_date FROM orders WHERE customer_id = c.customer_id LIMIT 1) AS recent;
-- ERROR: Unknown column 'c.customer_id' in 'where clause'

-- LATERAL fixes this
SELECT c.name, recent.order_date
FROM customers c
JOIN LATERAL (
    SELECT order_date
    FROM orders
    WHERE customer_id = c.customer_id
    ORDER BY order_date DESC
    LIMIT 1
) AS recent ON TRUE;
```

## Top-N per group

The most common use case: return the N most recent or highest-value rows for each group.

```sql
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name        VARCHAR(100)
);

CREATE TABLE orders (
    order_id    INT PRIMARY KEY,
    customer_id INT,
    order_date  DATE,
    total       DECIMAL(10,2)
);

-- Last 3 orders per customer
SELECT c.name, top_orders.order_date, top_orders.total
FROM customers c
JOIN LATERAL (
    SELECT order_date, total
    FROM orders
    WHERE customer_id = c.customer_id
    ORDER BY order_date DESC
    LIMIT 3
) AS top_orders ON TRUE
ORDER BY c.name, top_orders.order_date DESC;
```

## LEFT JOIN LATERAL to keep rows with no matches

Use `LEFT JOIN LATERAL` to keep customers who have no orders:

```sql
SELECT c.name, top_orders.order_date, top_orders.total
FROM customers c
LEFT JOIN LATERAL (
    SELECT order_date, total
    FROM orders
    WHERE customer_id = c.customer_id
    ORDER BY order_date DESC
    LIMIT 1
) AS top_orders ON TRUE;
```

Customers with no orders appear with `NULL` in the `order_date` and `total` columns.

## Computing per-row aggregates

```sql
-- Average of the top 5 orders per customer
SELECT c.name, top5.avg_top5
FROM customers c
JOIN LATERAL (
    SELECT ROUND(AVG(total), 2) AS avg_top5
    FROM (
        SELECT total
        FROM orders
        WHERE customer_id = c.customer_id
        ORDER BY total DESC
        LIMIT 5
    ) AS top_orders_sub
) AS top5 ON TRUE;
```

## Expanding a JSON or comma-separated column per row

`LATERAL` is useful for unnesting JSON arrays:

```sql
CREATE TABLE events (
    event_id INT PRIMARY KEY,
    tags     JSON
);

INSERT INTO events VALUES
(1, '["mysql","sql","database"]'),
(2, '["performance","index"]');

SELECT e.event_id, tag_value
FROM events e
JOIN LATERAL (
    SELECT jt.tag AS tag_value
    FROM JSON_TABLE(e.tags, '$[*]' COLUMNS (tag VARCHAR(50) PATH '$')) AS jt
) AS expanded ON TRUE;
```

## LATERAL vs correlated SELECT-list subquery

```sql
-- SELECT-list subquery (runs once per outer row, limited to scalar)
SELECT c.name,
       (SELECT MAX(total) FROM orders WHERE customer_id = c.customer_id) AS max_order
FROM customers c;

-- LATERAL (can return multiple columns and multiple rows per outer row)
SELECT c.name, lat.max_total, lat.order_count
FROM customers c
JOIN LATERAL (
    SELECT MAX(total) AS max_total, COUNT(*) AS order_count
    FROM orders
    WHERE customer_id = c.customer_id
) AS lat ON TRUE;
```

`LATERAL` is more flexible because it can return multiple columns without multiple correlated subqueries.

## Performance

`LATERAL` is executed once per row of the outer table, similar to a correlated subquery. Index the column used in the `WHERE` clause inside the lateral subquery:

```sql
CREATE INDEX idx_orders_customer ON orders (customer_id, order_date DESC);
```

Run `EXPLAIN` to verify index usage:

```sql
EXPLAIN
SELECT c.name, top_orders.order_date
FROM customers c
JOIN LATERAL (
    SELECT order_date FROM orders
    WHERE customer_id = c.customer_id
    ORDER BY order_date DESC LIMIT 1
) AS top_orders ON TRUE\G
```

## Summary

`LATERAL` derived tables in MySQL 8 allow subqueries in `FROM` to reference columns from preceding tables. The primary use cases are top-N-per-group queries, per-row multi-column aggregates, and JSON unnesting. Index the column referenced inside the lateral subquery to avoid full table scans, and use `LEFT JOIN LATERAL` when you need to preserve outer rows that have no matches.
