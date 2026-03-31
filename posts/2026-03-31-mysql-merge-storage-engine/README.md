# What Is the MERGE Storage Engine in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Storage Engine, MERGE, Partition, Table

Description: The MySQL MERGE storage engine creates a logical union of multiple identical MyISAM tables, allowing them to be queried as a single unified table.

---

## Overview

The MERGE storage engine (also called MRG_MyISAM) allows you to create a virtual table that logically combines multiple MyISAM tables with identical structure. Queries against the MERGE table scan all underlying MyISAM tables as if they were one. This provides a form of manual horizontal partitioning for MyISAM data.

MERGE was commonly used before MySQL's native `PARTITION BY` syntax was introduced, as a way to split large MyISAM tables across multiple physical files for performance or manageability. Today, it is largely superseded by InnoDB table partitioning.

## How MERGE Tables Work

All component tables must be MyISAM, must exist in the same database as the MERGE table, and must have identical column definitions and indexes.

```sql
-- Create two identical MyISAM tables for different years
CREATE TABLE orders_2024 (
  order_id INT NOT NULL AUTO_INCREMENT,
  customer_id INT NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  order_date DATE NOT NULL,
  PRIMARY KEY (order_id),
  INDEX idx_date (order_date)
) ENGINE=MyISAM;

CREATE TABLE orders_2025 (
  order_id INT NOT NULL AUTO_INCREMENT,
  customer_id INT NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  order_date DATE NOT NULL,
  PRIMARY KEY (order_id),
  INDEX idx_date (order_date)
) ENGINE=MyISAM;

-- Create the MERGE table pointing to both
CREATE TABLE orders_all (
  order_id INT NOT NULL AUTO_INCREMENT,
  customer_id INT NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  order_date DATE NOT NULL,
  PRIMARY KEY (order_id),
  INDEX idx_date (order_date)
) ENGINE=MERGE UNION=(orders_2024, orders_2025) INSERT_METHOD=LAST;
```

## Querying and Inserting

```sql
-- Query spans both underlying tables transparently
SELECT COUNT(*), SUM(amount)
FROM orders_all
WHERE order_date BETWEEN '2024-01-01' AND '2025-12-31';

-- INSERT_METHOD=LAST sends new inserts to orders_2025
INSERT INTO orders_all (customer_id, amount, order_date)
VALUES (101, 149.99, '2025-06-15');

-- INSERT_METHOD=FIRST would send inserts to orders_2024
-- INSERT_METHOD=NO (default) disables inserts via the MERGE table
```

## Changing the Union

You can alter the MERGE table to add or remove component tables without touching the data:

```sql
-- Add orders_2026 when it exists
CREATE TABLE orders_2026 (
  order_id INT NOT NULL AUTO_INCREMENT,
  customer_id INT NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  order_date DATE NOT NULL,
  PRIMARY KEY (order_id),
  INDEX idx_date (order_date)
) ENGINE=MyISAM;

ALTER TABLE orders_all UNION=(orders_2024, orders_2025, orders_2026);
```

## Limitations

MERGE tables have significant limitations compared to native partitioning:

- All component tables must be MyISAM - no InnoDB support
- No transaction support (MyISAM limitation)
- Duplicate keys can exist across component tables, causing issues
- Foreign keys are not supported
- `REPLACE` statements do not work correctly with MERGE tables

```sql
-- Check the MERGE table definition
SHOW CREATE TABLE orders_all\G
```

## Modern Alternative

For new development, use InnoDB's native partitioning instead of MERGE:

```sql
CREATE TABLE orders (
  order_id INT NOT NULL AUTO_INCREMENT,
  customer_id INT NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  order_date DATE NOT NULL,
  PRIMARY KEY (order_id, order_date)
) ENGINE=InnoDB
PARTITION BY RANGE (YEAR(order_date)) (
  PARTITION p2024 VALUES LESS THAN (2025),
  PARTITION p2025 VALUES LESS THAN (2026),
  PARTITION p2026 VALUES LESS THAN (2027)
);
```

## Summary

The MySQL MERGE storage engine logically unites multiple identical MyISAM tables into a single queryable surface. It was useful before native partitioning existed for manually splitting large MyISAM tables by time period or category. Today it is largely superseded by InnoDB partitioning, which provides the same logical grouping with full transaction support, foreign key enforcement, and better query optimization. MERGE remains available for legacy systems but should not be used in new designs.
