# How to Model Many-to-Many Relationships in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Many-to-Many, Schema Design, Array, Join, Analytics

Description: Learn how to model many-to-many relationships in ClickHouse using arrays, junction tables, and denormalization patterns suited to columnar analytics.

---

Many-to-many relationships (e.g., users have many roles, products have many tags) require special handling in ClickHouse because OLTP-style join tables hurt performance at scale. ClickHouse offers array-based and denormalized approaches that are more efficient.

## Traditional Junction Table Approach

While ClickHouse can use junction tables, they require joins that scan large amounts of data:

```sql
CREATE TABLE user_roles (
    user_id UInt64,
    role_id UInt32,
    assigned_at DateTime
) ENGINE = MergeTree()
ORDER BY (user_id, role_id);

-- Query: users with a specific role
SELECT DISTINCT u.user_id, u.email
FROM users u
JOIN user_roles ur ON u.user_id = ur.user_id
WHERE ur.role_id = 5;
```

This works but requires two table scans and a join.

## Preferred: Array Column Approach

Store the many side of the relationship as an array in the parent table:

```sql
CREATE TABLE users (
    user_id UInt64,
    email String,
    name String,
    roles Array(String),
    tags Array(String),
    created_at DateTime
) ENGINE = MergeTree()
ORDER BY user_id;

INSERT INTO users VALUES
    (1001, 'alice@example.com', 'Alice', ['admin', 'editor'], ['power-user'], now()),
    (1002, 'bob@example.com', 'Bob', ['viewer'], ['trial'], now());
```

Now you can query without joins:

```sql
-- Users with admin role
SELECT user_id, email
FROM users
WHERE has(roles, 'admin');

-- Count users per role
SELECT role, count() AS user_count
FROM users
ARRAY JOIN roles AS role
GROUP BY role
ORDER BY user_count DESC;
```

## Array-Based Many-to-Many with Bloom Filter Index

For high-cardinality arrays, add a bloom filter index for fast `has()` queries:

```sql
CREATE TABLE products (
    product_id UInt32,
    name String,
    tags Array(String),
    categories Array(UInt16)
) ENGINE = MergeTree()
ORDER BY product_id;

ALTER TABLE products
    ADD INDEX tags_bloom tags TYPE bloom_filter(0.01) GRANULARITY 1;

-- Efficient search
SELECT product_id, name
FROM products
WHERE has(tags, 'organic');
```

## Bi-Directional Many-to-Many

When you need to query from both sides, denormalize into both tables:

```sql
-- Orders table: each order has multiple products
CREATE TABLE orders (
    order_id String,
    customer_id UInt64,
    product_ids Array(UInt32),
    ordered_at DateTime
) ENGINE = MergeTree()
ORDER BY (customer_id, ordered_at);

-- Products table: each product has sales summary
CREATE TABLE product_sales_summary (
    product_id UInt32,
    total_orders UInt64,
    total_quantity UInt64,
    last_ordered_at DateTime
) ENGINE = SummingMergeTree(total_orders, total_quantity)
ORDER BY product_id;
```

## Summary of Patterns

```text
Pattern              | When to Use
---------------------|------------------------------------------------
Array column         | One side owns the relationship, read is primary
Junction table       | Frequent updates, complex filtering on both sides
Denormalized arrays  | Analytics, both sides queried, read-heavy
```

## Summary

ClickHouse handles many-to-many relationships most efficiently through array columns on the owning side of the relationship. Combined with `ARRAY JOIN` for explosion and bloom filter indexes for `has()` queries, this approach avoids the join overhead of junction tables and is well-suited to the append-only, read-heavy nature of analytical workloads.
