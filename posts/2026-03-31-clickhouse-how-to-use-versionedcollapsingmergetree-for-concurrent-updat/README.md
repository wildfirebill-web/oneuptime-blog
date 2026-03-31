# How to Use VersionedCollapsingMergeTree for Concurrent Updates

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, VersionedCollapsingMergeTree, Table Engine, Update, Concurrency

Description: Learn how to use VersionedCollapsingMergeTree in ClickHouse to handle concurrent state updates with version ordering, avoiding race conditions in distributed systems.

---

## Overview

`VersionedCollapsingMergeTree` extends `CollapsingMergeTree` by adding a version column to handle out-of-order inserts. In `CollapsingMergeTree`, sign rows must arrive in order (insert then cancel). With `VersionedCollapsingMergeTree`, rows with the same key and version can arrive in any order - ClickHouse uses the version number to determine which state is current.

This makes it ideal for distributed systems where updates may arrive out of order.

## Creating a VersionedCollapsingMergeTree Table

```sql
CREATE TABLE user_state
(
    user_id     UInt64,
    status      String,
    last_login  DateTime,
    version     UInt32,
    sign        Int8     -- +1 for insert, -1 for cancel
)
ENGINE = VersionedCollapsingMergeTree(sign, version)
ORDER BY user_id;
```

The engine takes two parameters: the sign column and the version column.

## Inserting Initial State

```sql
INSERT INTO user_state VALUES
(1001, 'active',   '2024-06-01 10:00:00', 1, 1),
(1002, 'inactive', '2024-05-15 09:00:00', 1, 1),
(1003, 'active',   '2024-06-01 12:00:00', 1, 1);
```

## Updating State (Cancel + Reinsert)

To update a row, insert the cancellation row (sign=-1 with the same version) and the new state row (sign=+1 with an incremented version):

```sql
-- Update user 1001: status changes from 'active' to 'suspended'
INSERT INTO user_state VALUES
-- Cancel old state
(1001, 'active',    '2024-06-01 10:00:00', 1, -1),
-- Insert new state
(1001, 'suspended', '2024-06-10 14:00:00', 2,  1);
```

## Out-of-Order Arrival (Key Advantage)

Unlike `CollapsingMergeTree`, the cancel row can arrive before the original insert:

```sql
-- Cancel arrives first (out of order)
INSERT INTO user_state VALUES
(1002, 'inactive', '2024-05-15 09:00:00', 1, -1);

-- Original insert arrives later - ClickHouse pairs them correctly by version
INSERT INTO user_state VALUES
(1002, 'inactive', '2024-05-15 09:00:00', 1, 1);
```

ClickHouse collapses these two rows (version=1, sign +1 and -1 cancel each other) regardless of arrival order.

## Querying Current State

Always use `sign` in queries until merges happen:

```sql
SELECT
    user_id,
    status,
    last_login
FROM user_state
WHERE sign = 1
ORDER BY user_id;
```

For more accurate results accounting for partially merged data:

```sql
SELECT
    user_id,
    argMax(status, version)     AS current_status,
    argMax(last_login, version) AS last_login_time
FROM user_state
GROUP BY user_id
HAVING sum(sign) > 0
ORDER BY user_id;
```

## Counting Current Records

```sql
SELECT count()
FROM user_state
GROUP BY user_id
HAVING sum(sign) = 1;

-- Or use FINAL to force merge-like behavior
SELECT count()
FROM user_state FINAL
WHERE sign = 1;
```

## How Collapsing Works

During background merges, ClickHouse pairs rows with:
- Same primary key (`user_id`)
- Same version number
- Opposite signs (+1 and -1)

These pairs are collapsed (deleted). Rows with unique versions and positive sign remain as the current state.

## Forcing a Merge for Testing

```sql
OPTIMIZE TABLE user_state FINAL;

-- After merge, only current-state rows remain
SELECT * FROM user_state ORDER BY user_id;
```

## Comparison with CollapsingMergeTree

| Feature | CollapsingMergeTree | VersionedCollapsingMergeTree |
|---|---|---|
| Out-of-order inserts | Problematic | Handled correctly |
| Distributed systems | Risky | Safe |
| Extra column needed | No | Yes (version) |
| Collapsing mechanism | By sign + arrival order | By sign + version |

## Summary

`VersionedCollapsingMergeTree` handles mutable state in ClickHouse with versioned rows, allowing out-of-order cancellation and reinsert pairs to arrive safely. Use it in distributed systems where update order cannot be guaranteed. Always query with `HAVING sum(sign) > 0` or use `argMax(column, version)` to retrieve current state before background merges complete. The version column ensures ClickHouse correctly pairs cancel (-1) and insert (+1) rows even when they arrive in arbitrary order.
