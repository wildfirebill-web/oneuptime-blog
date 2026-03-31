# How to Merge Partitions in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Partition, ALTER TABLE, InnoDB, Database Administration

Description: Learn how to merge multiple MySQL partitions into one using ALTER TABLE REORGANIZE PARTITION, and when merging partitions is the right maintenance strategy.

---

## What Does Merging Partitions Mean?

Merging partitions in MySQL means consolidating two or more existing partitions into a single partition. This is done using `ALTER TABLE ... REORGANIZE PARTITION`. There is no dedicated `MERGE PARTITION` command - reorganizing is the mechanism that covers both splitting and merging.

Merging is appropriate when:
- You have too many small historical partitions that are rarely queried
- You want to consolidate old date-range partitions into an archive partition
- Your partitioning strategy was over-granular and you want to simplify it

## Merge RANGE Partitions

Example: consolidate several year-based partitions into one archive partition.

```sql
-- Starting state: individual year partitions
CREATE TABLE transactions (
    txn_id BIGINT NOT NULL,
    txn_year INT NOT NULL,
    amount DECIMAL(12, 2),
    PRIMARY KEY (txn_id, txn_year)
)
PARTITION BY RANGE (txn_year)
(
    PARTITION p2019 VALUES LESS THAN (2020),
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024)
);
```

Merge 2019 through 2021 into a single archive partition:

```sql
ALTER TABLE transactions
REORGANIZE PARTITION p2019, p2020, p2021 INTO (
    PARTITION p_archive VALUES LESS THAN (2022)
);
```

MySQL reads all rows from the three source partitions and writes them into the single new partition. The boundaries must be contiguous - you cannot skip partitions when merging.

## Merge List Partitions

```sql
CREATE TABLE products (
    product_id INT NOT NULL,
    category_id INT NOT NULL,
    name VARCHAR(255),
    PRIMARY KEY (product_id, category_id)
)
PARTITION BY LIST (category_id)
(
    PARTITION p_electronics VALUES IN (1, 2),
    PARTITION p_computers VALUES IN (3, 4),
    PARTITION p_accessories VALUES IN (5, 6)
);

-- Merge electronics and computers into one tech partition
ALTER TABLE products
REORGANIZE PARTITION p_electronics, p_computers INTO (
    PARTITION p_tech VALUES IN (1, 2, 3, 4)
);
```

## Verify After Merging

```sql
SELECT
    PARTITION_NAME,
    PARTITION_DESCRIPTION,
    TABLE_ROWS,
    DATA_LENGTH
FROM INFORMATION_SCHEMA.PARTITIONS
WHERE TABLE_SCHEMA = 'mydb'
  AND TABLE_NAME = 'transactions'
ORDER BY PARTITION_ORDINAL_POSITION;
```

The merged partitions should be gone and replaced by the new single partition.

## Validate Row Count Preservation

Before and after row counts should match:

```sql
-- Before merge: count rows in affected partitions
SELECT COUNT(*) FROM transactions
WHERE txn_year < 2022;

-- After merge: same count from the archive partition
SELECT COUNT(*) FROM transactions
PARTITION (p_archive);
```

## Constraints for Merging

- You can only merge adjacent partitions (for RANGE) or logically related partitions (for LIST)
- The new partition's `VALUES LESS THAN` or `VALUES IN` must exactly cover all rows from the merged partitions
- You cannot merge HASH or KEY partitions this way; use `COALESCE PARTITION` for those
- Merging large partitions is I/O intensive and may take significant time

## Performance Tip

If the partitions being merged are large and the operation is too slow, consider:
1. Using `pt-online-schema-change` or `gh-ost` for online schema changes
2. Running the operation during low-traffic windows
3. Merging in smaller batches rather than all at once

## Summary

Partition merging in MySQL is performed via `REORGANIZE PARTITION` by specifying multiple source partitions and a single destination partition. It is the right approach for consolidating old historical partitions to reduce management overhead, and it guarantees no data is lost during the operation.
