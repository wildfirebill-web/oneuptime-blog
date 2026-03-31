# What Is a MySQL Secondary Index

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Secondary Index, InnoDB

Description: Learn what a MySQL secondary index is, how InnoDB stores secondary indexes with primary key values, and when secondary indexes improve query performance.

---

A secondary index in MySQL is any index that is not the primary key (clustered) index. Secondary indexes provide additional access paths to rows in a table, enabling MySQL to quickly locate rows that match filter or sort conditions on non-primary-key columns.

## How Secondary Indexes Work in InnoDB

In InnoDB, secondary indexes are separate B-trees from the clustered (primary key) index. Each leaf node of a secondary index contains:
- The indexed column value(s)
- The primary key value(s) of the matching row

This means that reading a full row through a secondary index requires two B-tree traversals: one on the secondary index to find the primary key, then one on the clustered index to fetch the full row data.

```sql
-- Table with primary key and secondary index
CREATE TABLE orders (
  id          INT AUTO_INCREMENT PRIMARY KEY,
  customer_id INT NOT NULL,
  status      VARCHAR(20),
  total       DECIMAL(10,2),
  INDEX idx_customer (customer_id)
) ENGINE=InnoDB;

-- Query using secondary index:
-- Step 1: scan idx_customer B-tree for customer_id = 42
-- Step 2: for each match, look up id in the clustered index
EXPLAIN SELECT * FROM orders WHERE customer_id = 42\G
-- key: idx_customer
-- Extra: Using index condition (or none for full row fetch)
```

## Creating Secondary Indexes

```sql
-- Single-column secondary index
CREATE INDEX idx_status ON orders (status);

-- Composite secondary index
CREATE INDEX idx_cust_status ON orders (customer_id, status);

-- Index during table creation
CREATE TABLE products (
  id    INT AUTO_INCREMENT PRIMARY KEY,
  sku   VARCHAR(50) NOT NULL,
  price DECIMAL(10,2),
  INDEX idx_price (price)
) ENGINE=InnoDB;
```

## When Secondary Indexes Are Used

MySQL's optimizer decides whether to use a secondary index based on selectivity, table size, and query structure.

```sql
-- High selectivity: optimizer uses the index
SELECT * FROM orders WHERE customer_id = 42;
-- Matches few rows - index is efficient

-- Low selectivity: optimizer may skip the index (full scan faster)
SELECT * FROM orders WHERE status = 'completed';
-- If 80% of rows have status='completed', full scan beats index lookup
```

Use `EXPLAIN` to verify index usage:

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending'\G
-- key: idx_cust_status
-- rows: 3
```

## Secondary Index Overhead on Writes

Every secondary index must be updated on INSERT, UPDATE, and DELETE. Tables with many secondary indexes have slower write performance.

```sql
-- Too many indexes slow down writes
CREATE TABLE events (
  id       BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id  INT,
  type     VARCHAR(50),
  severity VARCHAR(20),
  data     TEXT,
  INDEX idx1 (user_id),
  INDEX idx2 (type),
  INDEX idx3 (severity),
  INDEX idx4 (user_id, type),
  INDEX idx5 (type, severity)
  -- Five secondary indexes: every INSERT updates all five
) ENGINE=InnoDB;
```

Keep secondary indexes to those actually used by production queries.

## Monitoring Secondary Index Usage

```sql
-- Find unused secondary indexes (MySQL 8.0+)
SELECT object_schema, object_name, index_name
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
  AND index_name != 'PRIMARY'
  AND count_star = 0
  AND object_schema = DATABASE()
ORDER BY object_name, index_name;
```

## Secondary Index Size

Secondary indexes store the index values plus the primary key, so they grow with table size. Monitor index sizes to plan capacity.

```sql
SELECT
  TABLE_NAME,
  INDEX_NAME,
  ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE database_name = DATABASE()
  AND stat_name = 'size'
ORDER BY size_mb DESC;
```

## Summary

Secondary indexes in InnoDB are separate B-trees that store indexed column values alongside the primary key. They enable fast lookups on non-PK columns but require a second lookup against the clustered index to fetch full row data (unless a covering index is used). Each secondary index adds write overhead - only create indexes that are regularly used by production queries. Monitor index usage with `performance_schema` to identify and drop unused indexes.
