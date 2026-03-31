# How to Handle Data Corrections in Immutable ClickHouse Tables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Data Correction, Mutation, ReplacingMergeTree, Immutability

Description: Learn practical strategies for correcting data in ClickHouse's immutable append-only storage, including mutations, versioning, and correction patterns.

---

## The Immutability Challenge

ClickHouse is fundamentally append-only. Parts are immutable once written. When data errors occur - wrong values ingested, duplicate events, misrouted records - correction requires one of several strategies depending on the severity and frequency of corrections.

## Strategy 1: Heavy Mutations (UPDATE/DELETE)

For small, targeted corrections, use ALTER TABLE mutations:

```sql
-- Correct a specific wrong value
ALTER TABLE orders
    UPDATE amount = 99.99
    WHERE order_id = 12345 AND amount = 9.99;

-- Remove a duplicated event
ALTER TABLE events
    DELETE WHERE event_id = 'duplicate-event-id';
```

Mutations are expensive - they rewrite all affected parts. Use them sparingly for one-off corrections, not bulk updates.

Monitor progress:

```sql
SELECT command, parts_to_do, is_done
FROM system.mutations
WHERE table = 'orders' AND is_done = 0;
```

## Strategy 2: Correction Pattern with ReplacingMergeTree

Design the table upfront for corrections using ReplacingMergeTree:

```sql
CREATE TABLE transactions (
    transaction_id  String,
    amount          Decimal(18, 2),
    currency        LowCardinality(String),
    version         UInt64 DEFAULT 1
) ENGINE = ReplacingMergeTree(version)
ORDER BY transaction_id;
```

To correct a transaction, insert a new row with a higher version:

```sql
-- Original (wrong)
INSERT INTO transactions VALUES ('txn-001', 9.99, 'USD', 1);

-- Correction (right)
INSERT INTO transactions VALUES ('txn-001', 99.99, 'USD', 2);
```

Query with FINAL to get the latest version:

```sql
SELECT transaction_id, amount, currency
FROM transactions FINAL
WHERE transaction_id = 'txn-001';
```

## Strategy 3: Explicit Correction Log Table

Maintain a separate corrections table to track what was wrong and what the correct value should be:

```sql
CREATE TABLE corrections (
    target_table    String,
    primary_key     String,
    field_name      String,
    old_value       String,
    new_value       String,
    corrected_at    DateTime,
    corrected_by    String
) ENGINE = MergeTree()
ORDER BY (corrected_at, target_table);
```

Application queries JOIN the corrections table to apply corrections at read time, avoiding mutations entirely.

## Strategy 4: CollapsingMergeTree for Precise Cancellations

```sql
CREATE TABLE inventory (
    item_id     UInt64,
    quantity    Int32,
    sign        Int8
) ENGINE = CollapsingMergeTree(sign)
ORDER BY item_id;

-- Original
INSERT INTO inventory VALUES (1, 100, 1);

-- Cancel and correct
INSERT INTO inventory VALUES (1, 100, -1);
INSERT INTO inventory VALUES (1, 95, 1);
```

## Partition-Level Correction

For large-scale corrections affecting an entire time period, reprocess and replace a partition:

```sql
-- Reprocess corrected data into a staging table
INSERT INTO orders_corrected_staging
SELECT corrected_data FROM raw_source WHERE toYYYYMM(created_at) = 202401;

-- Replace partition atomically
ALTER TABLE orders REPLACE PARTITION '202401' FROM orders_corrected_staging;
```

This is the most efficient bulk correction approach.

## Summary

Data corrections in ClickHouse range from lightweight ReplacingMergeTree inserts for row-level overwrites to partition-level replacement for bulk corrections. Use heavy mutations only for small-scale targeted corrections, and design tables with correction in mind from the start by incorporating version columns and appropriate merge tree engines.
