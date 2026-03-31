# How to Denormalize a MySQL Database for Performance

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Denormalization, Performance, Schema Design, Optimization

Description: Learn when and how to denormalize a MySQL database to improve read performance by adding redundant data, precomputed values, and flattened structures.

---

## What Is Denormalization?

Denormalization is the deliberate introduction of redundancy into a normalized schema to improve read performance. Normalization reduces redundancy at the cost of joins; denormalization reduces joins at the cost of storage and write complexity.

Use denormalization after profiling shows that join-heavy queries are a real bottleneck - not as a first step.

## When to Denormalize

- Read-heavy workloads where the same joined result is needed millions of times per day.
- Reporting and analytics queries that aggregate across many tables.
- The cost of extra writes and synchronization logic is acceptable.
- Caching alone does not provide enough relief.

## Technique 1: Add a Redundant Column

Store frequently needed data in the same table to avoid a join:

```sql
-- Normalized: orders has no customer name
CREATE TABLE orders (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    customer_id INT UNSIGNED NOT NULL,
    total       DECIMAL(10,2) NOT NULL
);

-- Denormalized: snapshot the customer name at order time
ALTER TABLE orders ADD COLUMN customer_name VARCHAR(100) NOT NULL DEFAULT '';
```

```sql
-- Update the snapshot on insert (application code)
INSERT INTO orders (customer_id, total, customer_name)
SELECT 42, 99.99, name FROM customers WHERE id = 42;
```

The trade-off: if a customer changes their name, historical orders are unaffected (often correct behavior for receipts).

## Technique 2: Precomputed Aggregates

Store aggregate values to avoid expensive `COUNT` or `SUM` queries:

```sql
ALTER TABLE products ADD COLUMN review_count INT UNSIGNED NOT NULL DEFAULT 0;
ALTER TABLE products ADD COLUMN average_rating DECIMAL(3,2) NOT NULL DEFAULT 0.00;
```

Maintain them with triggers:

```sql
DELIMITER $$
CREATE TRIGGER after_review_insert
AFTER INSERT ON reviews FOR EACH ROW
BEGIN
    UPDATE products
    SET
        review_count   = review_count + 1,
        average_rating = (average_rating * (review_count - 1) + NEW.rating) / review_count
    WHERE id = NEW.product_id;
END$$
DELIMITER ;
```

## Technique 3: Flatten a Many-to-Many Join Table

For read-heavy tag lookup, denormalize into a JSON column:

```sql
ALTER TABLE articles ADD COLUMN tag_list JSON;

-- Populate with a periodic job
UPDATE articles a
SET tag_list = (
    SELECT JSON_ARRAYAGG(t.name)
    FROM article_tags at
    JOIN tags t ON t.id = at.tag_id
    WHERE at.article_id = a.id
);
```

Query without joins:

```sql
SELECT id, title FROM articles WHERE JSON_CONTAINS(tag_list, '"mysql"');
```

## Technique 4: Summary Tables

Create a pre-aggregated table updated periodically or via events:

```sql
CREATE TABLE daily_sales_summary (
    sale_date    DATE         NOT NULL PRIMARY KEY,
    total_orders INT UNSIGNED NOT NULL DEFAULT 0,
    total_revenue DECIMAL(12,2) NOT NULL DEFAULT 0.00
);

-- Refresh nightly
INSERT INTO daily_sales_summary (sale_date, total_orders, total_revenue)
SELECT DATE(created_at), COUNT(*), SUM(total)
FROM orders
WHERE DATE(created_at) = CURDATE() - INTERVAL 1 DAY
ON DUPLICATE KEY UPDATE
    total_orders   = VALUES(total_orders),
    total_revenue  = VALUES(total_revenue);
```

## Keeping Denormalized Data Consistent

Options for synchronization:

```text
1. Application logic - update all copies at write time.
2. Database triggers - automatic but can hide complexity.
3. Scheduled jobs - periodic refresh, accepts some staleness.
4. Event streaming - CDC tools like Debezium propagate changes asynchronously.
```

## Summary

Denormalize strategically when profiling confirms that joins are the bottleneck. Common techniques include redundant columns for snapshot data, precomputed aggregates maintained by triggers, JSON tag lists for flat lookups, and summary tables for reporting queries. Always document each piece of redundant data and the mechanism used to keep it consistent.
