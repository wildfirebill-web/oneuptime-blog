# How to Use Nested Data Type in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, Data Type, Nested

Description: Learn how ClickHouse stores Nested columns as parallel arrays, how to insert and query nested data, and how to use ARRAY JOIN to expand rows.

---

`Nested(col1 T1, col2 T2, ...)` lets you define a sub-table structure within a column. Internally, ClickHouse stores each sub-column as a separate `Array(T)` column on disk. This means each sub-column is individually compressed and can be read independently - making Nested a good fit for repeating groups of structured data with a fixed schema, such as order line items, log event attributes, or time-series samples.

## Defining Nested Columns

Declare a Nested structure inside a `CREATE TABLE` statement. All sub-column arrays for a given row must have the same length.

```sql
CREATE TABLE orders (
    order_id    UInt64,
    customer_id UInt32,
    items Nested(
        product_id   UInt32,
        name         String,
        quantity     UInt16,
        unit_price   Float64
    ),
    created_at DateTime
) ENGINE = MergeTree()
ORDER BY (customer_id, order_id);
```

## Inserting Nested Data

Insert Nested values as parallel arrays - each sub-column is an array of the same length.

```sql
INSERT INTO orders VALUES
(
    1001,
    42,
    [101, 102, 103],          -- items.product_id
    ['Widget A', 'Widget B', 'Gadget X'],  -- items.name
    [2, 1, 3],                -- items.quantity
    [9.99, 24.99, 14.95],     -- items.unit_price
    '2026-03-15 14:00:00'
),
(
    1002,
    43,
    [102],
    ['Widget B'],
    [5],
    [24.99],
    '2026-03-15 15:30:00'
);
```

## Accessing Nested Sub-Columns

Access a sub-column using the `column.sub_column` dot notation. The result is an `Array(T)`.

```sql
-- Read all product names for each order
SELECT order_id, items.name AS product_names FROM orders;

-- Access a specific element by index (1-based)
SELECT order_id, items.name[1] AS first_item FROM orders;

-- Calculate order total
SELECT
    order_id,
    arrayReduce(
        'sum',
        arrayMap((q, p) -> q * p, items.quantity, items.unit_price)
    ) AS order_total
FROM orders;
```

## ARRAY JOIN with Nested

`ARRAY JOIN` on a Nested column expands all sub-columns in sync, so each row represents a single nested element.

```sql
-- Expand each order into individual line items
SELECT
    order_id,
    customer_id,
    items.product_id AS product_id,
    items.name       AS product_name,
    items.quantity   AS qty,
    items.unit_price AS price,
    items.quantity * items.unit_price AS line_total
FROM orders
ARRAY JOIN items;

-- Filter to a specific product across all orders
SELECT order_id, items.quantity AS qty
FROM orders
ARRAY JOIN items
WHERE items.product_id = 102;
```

## Aggregating Nested Data

After expanding with `ARRAY JOIN`, standard aggregations apply normally.

```sql
-- Best-selling products by total quantity
SELECT
    items.product_id AS product_id,
    items.name       AS product_name,
    sum(items.quantity) AS total_sold
FROM orders
ARRAY JOIN items
GROUP BY product_id, product_name
ORDER BY total_sold DESC;
```

## Limitations of Nested

Nested has several important limitations to be aware of before choosing it over alternatives.

```sql
-- You cannot use Nested inside another Nested (no nesting of Nested)
-- You cannot use ALTER TABLE to add a column inside an existing Nested
--   without recreating or using specific ClickHouse versions

-- Limitation example: check sub-column length consistency at insert time
-- The following would fail because array lengths don't match:
--   INSERT INTO orders VALUES (9999, 1, [101, 102], ['Widget A'], [1], [9.99], now());
-- Error: Sizes of nested columns do not match

-- Alternative for dynamic schemas: use Map(String, String) or
-- separate child tables joined by order_id
```

## Summary

`Nested(col1 T1, col2 T2, ...)` provides a convenient way to embed structured repeating data within a row, with each sub-column stored as a separate compressed array for efficient columnar access. Use `ARRAY JOIN` to expand nested rows for aggregation and filtering. Be aware of the equal-length constraint across sub-columns at insert time and the restriction against nesting Nested types - for dynamic or deeply hierarchical structures, `Map` or separate tables are better alternatives.
