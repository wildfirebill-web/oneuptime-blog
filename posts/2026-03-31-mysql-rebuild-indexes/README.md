# How to Rebuild Indexes in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Index, InnoDB, Maintenance, Performance

Description: Learn how to rebuild MySQL indexes to reclaim fragmented space, update statistics, and improve query performance using OPTIMIZE TABLE and ALTER TABLE FORCE.

---

## Why Rebuild Indexes?

Over time, MySQL InnoDB indexes can become fragmented as rows are inserted, updated, and deleted. Fragmentation leaves gaps in B-tree pages, increasing disk I/O and reducing cache efficiency. Rebuilding indexes packs the B-tree pages and updates cardinality statistics.

## Using OPTIMIZE TABLE

`OPTIMIZE TABLE` rebuilds the table and all its indexes, defragments the data, and reclaims unused space:

```sql
OPTIMIZE TABLE orders;
```

```text
+------------------+----------+----------+-------------------------------------------------------------------+
| Table            | Op       | Msg_type | Msg_text                                                          |
+------------------+----------+----------+-------------------------------------------------------------------+
| mydb.orders      | optimize | note     | Table does not support optimize, doing recreate + analyze instead |
| mydb.orders      | optimize | status   | OK                                                                |
+------------------+----------+----------+-------------------------------------------------------------------+
```

For InnoDB tables, MySQL performs a table rebuild (`ALTER TABLE ... FORCE`) followed by `ANALYZE TABLE`.

## Using ALTER TABLE FORCE

`ALTER TABLE ... FORCE` triggers a full table rebuild - equivalent to `OPTIMIZE TABLE` for InnoDB but with explicit online control:

```sql
ALTER TABLE orders FORCE, ALGORITHM=INPLACE, LOCK=NONE;
```

This rebuilds the clustered index and all secondary indexes without blocking reads or writes.

## Rebuilding a Single Index

MySQL does not provide a command to rebuild one index in isolation. You must drop and re-add it:

```sql
-- Drop and re-add to rebuild
ALTER TABLE orders
    DROP INDEX idx_customer_id,
    ADD INDEX idx_customer_id (customer_id),
    ALGORITHM = INPLACE,
    LOCK = NONE;
```

## Checking Fragmentation Before Rebuilding

Use the InnoDB metrics to see how fragmented a table is:

```sql
SELECT
    TABLE_NAME,
    DATA_LENGTH,
    INDEX_LENGTH,
    DATA_FREE,
    ROUND(DATA_FREE / (DATA_LENGTH + INDEX_LENGTH) * 100, 1) AS frag_pct
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database'
ORDER BY frag_pct DESC;
```

`DATA_FREE` shows unused space inside the tablespace. A `frag_pct` above 20-30% is a reasonable threshold for rebuilding.

## Refreshing Statistics Without a Full Rebuild

If the goal is only to update cardinality estimates without a full rebuild:

```sql
ANALYZE TABLE orders;
```

`ANALYZE TABLE` is much faster than `OPTIMIZE TABLE` and does not modify the physical data layout.

## Scheduling Maintenance

For large tables, schedule rebuilds during low-traffic windows or use `pt-online-schema-change` from Percona Toolkit to rebuild with minimal impact:

```bash
pt-online-schema-change \
  --alter "ENGINE=InnoDB" \
  --execute \
  D=your_database,t=orders
```

## After Rebuilding

Verify the result by checking fragmentation again and reviewing cardinality:

```sql
SELECT DATA_FREE FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_database' AND TABLE_NAME = 'orders';

SHOW INDEX FROM orders;
```

## Summary

Rebuild MySQL indexes with `OPTIMIZE TABLE` or `ALTER TABLE ... FORCE, ALGORITHM=INPLACE, LOCK=NONE` when fragmentation is high (DATA_FREE ratio above 20-30%). Use `ANALYZE TABLE` when you only need fresh cardinality statistics. For production environments with large tables, prefer `ALGORITHM=INPLACE, LOCK=NONE` or Percona's `pt-online-schema-change` to minimize disruption.
