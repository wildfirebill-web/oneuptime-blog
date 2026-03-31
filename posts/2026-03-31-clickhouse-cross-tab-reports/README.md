# How to Create Cross-Tab Reports in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cross-Tab, Pivot, Analytics, Conditional Aggregation

Description: Learn how to create cross-tab (pivot) reports in ClickHouse using conditional aggregation to turn row values into column headers dynamically.

---

## What is a Cross-Tab Report?

A cross-tab report (pivot table) transforms distinct values from one column into separate columns, with aggregate values at each intersection. ClickHouse does not have a native PIVOT syntax, but conditional aggregation achieves the same result efficiently.

## Sample Sales Data

```sql
CREATE TABLE sales
(
    sale_date Date,
    region String,
    product String,
    amount Float64
)
ENGINE = MergeTree()
ORDER BY (sale_date, region);

INSERT INTO sales VALUES
('2024-01-15', 'North', 'Widget', 500),
('2024-01-15', 'South', 'Widget', 300),
('2024-01-15', 'North', 'Gadget', 700),
('2024-01-16', 'South', 'Gadget', 400),
('2024-01-16', 'East', 'Widget', 200);
```

## Basic Cross-Tab with Conditional Aggregation

Pivot products into columns:

```sql
SELECT
    sale_date,
    region,
    sumIf(amount, product = 'Widget') AS widget_sales,
    sumIf(amount, product = 'Gadget') AS gadget_sales
FROM sales
GROUP BY sale_date, region
ORDER BY sale_date, region;
```

## Cross-Tab with Multiple Metrics

Show both count and sum per product:

```sql
SELECT
    region,
    countIf(product = 'Widget') AS widget_orders,
    sumIf(amount, product = 'Widget') AS widget_revenue,
    countIf(product = 'Gadget') AS gadget_orders,
    sumIf(amount, product = 'Gadget') AS gadget_revenue
FROM sales
GROUP BY region
ORDER BY region;
```

## Adding Row Totals

```sql
SELECT
    region,
    sumIf(amount, product = 'Widget') AS widget_sales,
    sumIf(amount, product = 'Gadget') AS gadget_sales,
    sum(amount) AS total_sales
FROM sales
GROUP BY region
WITH TOTALS
ORDER BY region;
```

## Dynamic Cross-Tab via groupArray

When columns are not known in advance, use `groupArray` to collect values as an array:

```sql
SELECT
    region,
    groupArray(product) AS products,
    groupArray(amount) AS amounts
FROM sales
GROUP BY region;
```

Then post-process in your application layer, or use `arrayZip`:

```sql
SELECT
    region,
    arrayMap((p, a) -> concat(p, ':', toString(a)),
        groupArray(product),
        groupArray(amount)
    ) AS product_amounts
FROM sales
GROUP BY region;
```

## Percentage Distribution Across Columns

```sql
SELECT
    region,
    sumIf(amount, product = 'Widget') / sum(amount) * 100 AS widget_pct,
    sumIf(amount, product = 'Gadget') / sum(amount) * 100 AS gadget_pct,
    sum(amount) AS total
FROM sales
GROUP BY region
ORDER BY total DESC;
```

## Summary

Cross-tab reports in ClickHouse rely on `sumIf`, `countIf`, and `avgIf` to pivot row data into columns. Add `WITH TOTALS` for grand totals and use `arrayZip` with `groupArray` when the column list is dynamic.
