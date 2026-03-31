# How ClickHouse MVCC Works for Concurrent Access

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MVCC, Concurrency, Isolation, MergeTree

Description: Explains how ClickHouse handles concurrent reads and writes without traditional MVCC, using immutable data parts and background merges to provide isolation.

---

## ClickHouse Is Not Traditional MVCC

PostgreSQL and MySQL implement Multi-Version Concurrency Control (MVCC) with row-level version chains that allow readers and writers to access different versions simultaneously. ClickHouse does not use MVCC in the traditional sense. Instead, it achieves read-write isolation through immutable data parts.

## Immutable Parts: The Concurrency Model

When you insert data into a MergeTree table:
1. ClickHouse writes a new, immutable data part on disk
2. The part is immediately visible to subsequent queries
3. Background merges combine small parts into larger ones
4. The old parts remain readable until a merge creates a replacement part and the old parts are no longer referenced

This means:
- Reads never block writes
- Writes never block reads
- Merges and reads can proceed simultaneously

```sql
-- This query can run safely while inserts are happening
SELECT user_id, count()
FROM events
WHERE event_time >= today()
GROUP BY user_id;

-- This insert does not block the query above
INSERT INTO events VALUES (1, 42, 'click', now());
```

## Parts Visibility

```sql
-- See currently active parts
SELECT name, active, rows, bytes_on_disk
FROM system.parts
WHERE table = 'events'
  AND database = 'analytics'
ORDER BY min_time DESC
LIMIT 20;
```

Active parts are visible to queries. Inactive parts (being replaced by a merge) are still held until in-flight queries release them.

## Limitations vs True MVCC

ClickHouse does not support:
- **Row-level transactions** - each INSERT is atomic, but multi-statement transactions affecting the same rows are not supported in the traditional sense
- **Read-your-writes consistency** - an insert is visible immediately, but replicated writes may lag
- **Snapshot isolation for long transactions** - there is no "transaction start" snapshot in standard MergeTree

## INSERT Atomicity

Each `INSERT` statement is atomic at the part level. Either the part is written successfully and becomes visible, or it is rolled back entirely.

```sql
-- This insert is atomic: all rows land in one part or none do
INSERT INTO events (event_id, user_id, event_time)
SELECT number, number % 1000, now()
FROM numbers(100000);
```

## Handling Duplicates

ClickHouse generates a deterministic block hash for each INSERT. If a server crashes mid-insert and the client retries, the duplicate block is detected and silently ignored (controlled by `insert_deduplicate` setting).

```sql
-- Check deduplication behavior
SHOW CREATE TABLE events;
-- MergeTree tables have insert_deduplicate enabled by default in replicated mode
```

## Read After Write Consistency

On a single node, an INSERT is immediately visible to subsequent SELECTs. In a replicated cluster, there may be a replication lag. Use `SYSTEM SYNC REPLICA` to force consistency.

```sql
-- Wait for all replicas to catch up before querying
SYSTEM SYNC REPLICA events;
SELECT count() FROM events WHERE event_time = today();
```

## Summary

ClickHouse achieves concurrent access through immutable data parts rather than traditional MVCC. Reads and writes never block each other because each INSERT creates a new part that is immediately visible. Background merges operate on parts independently of active queries. This model provides atomic inserts and strong read-write isolation but does not support multi-statement transactions or snapshot isolation across long-running reads.
