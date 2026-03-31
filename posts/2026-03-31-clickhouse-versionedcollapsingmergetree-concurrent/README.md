# How to Handle Concurrent State Updates with VersionedCollapsingMergeTree in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, VersionedCollapsingMergeTree, Concurrency, State Updates, Deduplication, Versioning

Description: Use ClickHouse VersionedCollapsingMergeTree to safely handle out-of-order and concurrent state updates by combining sign and version columns.

---

`CollapsingMergeTree` requires that canceling rows arrive before the rows they cancel, which breaks down when inserts arrive out of order from concurrent producers. `VersionedCollapsingMergeTree` solves this by adding a version column - ClickHouse collapses pairs of rows with the same primary key and version regardless of insertion order.

## Table Definition

```sql
CREATE TABLE order_state
(
    order_id    UInt64,
    user_id     UInt64,
    status      LowCardinality(String),
    total_usd   Decimal(10, 2),
    updated_at  DateTime,
    version     UInt64,
    sign        Int8
)
ENGINE = VersionedCollapsingMergeTree(sign, version)
PARTITION BY toYYYYMM(updated_at)
ORDER BY (order_id);
```

The `sign` column is `1` for current state and `-1` for the cancellation of a previous state. The `version` column must be strictly increasing (use a Unix timestamp in milliseconds or a monotonic counter).

## Insert Initial State

```sql
INSERT INTO order_state VALUES
(5001, 42, 'pending', 99.99, now(), toUnixTimestamp64Milli(now64()), 1);
```

## Update State: Write Cancellation + New Version Concurrently

With `VersionedCollapsingMergeTree`, both the cancellation and the new row can be written in any order:

```sql
-- These two inserts can arrive in either order; ClickHouse will collapse them correctly
-- New state (version = current timestamp in ms)
INSERT INTO order_state VALUES
(5001, 42, 'paid', 99.99, now(), 1743379200001, 1);

-- Cancellation of version 1743379200000
INSERT INTO order_state VALUES
(5001, 42, 'pending', 99.99, '2026-03-31 10:00:00', 1743379200000, -1);
```

ClickHouse will match and collapse pairs with the same `(order_id, version)` regardless of which arrived first.

## Query Current State (Before Merge)

```sql
SELECT
    order_id,
    user_id,
    argMaxIf(status, version, sign = 1)       AS current_status,
    argMaxIf(total_usd, version, sign = 1)    AS current_total
FROM order_state
GROUP BY order_id, user_id
HAVING sum(sign) > 0;
```

## Query Using FINAL

```sql
SELECT order_id, user_id, status, total_usd
FROM order_state FINAL
WHERE status = 'pending'
ORDER BY order_id;
```

## Simulate Concurrent Producers

```python
import clickhouse_connect
import time
import threading

client = clickhouse_connect.get_client(host='localhost')

def update_order(order_id, old_status, new_status, old_version):
    new_version = int(time.time() * 1000)
    # Cancel old
    client.insert('order_state',
        [[order_id, 42, old_status, 99.99, '2026-03-31 10:00:00', old_version, -1]],
        column_names=['order_id','user_id','status','total_usd','updated_at','version','sign'])
    # Insert new
    client.insert('order_state',
        [[order_id, 42, new_status, 99.99, '2026-03-31 10:00:01', new_version, 1]],
        column_names=['order_id','user_id','status','total_usd','updated_at','version','sign'])

# Two threads updating different orders simultaneously
t1 = threading.Thread(target=update_order, args=(5001, 'pending', 'paid', 1743379200000))
t2 = threading.Thread(target=update_order, args=(5002, 'pending', 'paid', 1743379200000))
t1.start(); t2.start()
t1.join(); t2.join()
```

## Force Collapse and Verify

```sql
OPTIMIZE TABLE order_state FINAL;

-- Should show only the latest version per order
SELECT order_id, status, version, sign
FROM order_state
ORDER BY order_id, version;
```

## Comparison: CollapsingMergeTree vs VersionedCollapsingMergeTree

| Feature | CollapsingMergeTree | VersionedCollapsingMergeTree |
|---|---|---|
| Row ordering required | Yes (cancel before insert) | No (order-independent) |
| Concurrent producers safe | No | Yes |
| Extra column needed | sign only | sign + version |
| Background collapse | Yes | Yes |

## Summary

`VersionedCollapsingMergeTree` is the right choice when state updates arrive from multiple concurrent producers or when you cannot guarantee insertion order. By pairing a monotonically increasing version with the sign column, ClickHouse can collapse stale rows during background merges regardless of which arrived first, giving you correct current-state queries without locking or coordination.
