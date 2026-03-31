# When to Use Log Family Engines vs MergeTree in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Log Engine, MergeTree, Table Engine, TinyLog, StripeLog, Performance

Description: Learn when to choose Log family engines (Log, TinyLog, StripeLog) over MergeTree in ClickHouse based on table size, query patterns, and operational requirements.

---

ClickHouse offers two major categories of table engines for on-disk storage: the **Log family** (TinyLog, StripeLog, Log) and the **MergeTree family** (MergeTree, ReplacingMergeTree, SummingMergeTree, etc.). The right choice depends on table size, query frequency, concurrency requirements, and whether you need indexing, partitioning, or TTL.

## Log Family at a Glance

```text
Engine     Max Size    Concurrent Reads  Index  Partitioning  TTL  Use Case
TinyLog    ~1 MB       No                No     No            No   Tiny temp tables
StripeLog  ~1 GB       Yes               No     No            No   Small lookup tables
Log        ~1 GB       Yes               No     No            No   Small multi-read tables
```

## MergeTree Family at a Glance

```text
Engine               Partition  Index  TTL  Dedup  Best For
MergeTree            Yes        Yes    Yes  No     Append-only facts
ReplacingMergeTree   Yes        Yes    Yes  Yes    Mutable records (CDC)
SummingMergeTree     Yes        Yes    Yes  Sum    Pre-aggregated counters
AggregatingMergeTree Yes        Yes    Yes  Any    Materialized view backing
```

## Choose Log Family When

1. **Table is tiny (under 1 GB)** - Log engines avoid MergeTree's compaction overhead.
2. **No filtering needed** - No WHERE clause optimization or index required.
3. **Simple write-once, read-many** - Audit logs, configuration tables, seed data.
4. **Development or testing** - Lightweight fixtures without background processes.

```sql
-- Good use of Log: tiny country lookup
CREATE TABLE country_codes (
    code String,
    name String
) ENGINE = Log;

INSERT INTO country_codes VALUES ('US', 'United States'), ('DE', 'Germany');
```

## Choose MergeTree When

1. **Table is large (over 1 GB or billions of rows)** - Index and partitioning are essential.
2. **Queries filter on specific columns** - Primary index and skip indexes enable pruning.
3. **Time-based retention needed** - TTL expressions delete or move old data automatically.
4. **High-frequency inserts** - Background merges compact parts for efficient reads.
5. **Mutations needed** - ALTER TABLE ... UPDATE/DELETE (mutations) work only on MergeTree.

```sql
-- Correct use of MergeTree: billions of events
CREATE TABLE events (
    ts      DateTime,
    user_id UInt64,
    event   String
) ENGINE = MergeTree
PARTITION BY toYYYYMM(ts)
ORDER BY (ts, user_id)
TTL ts + INTERVAL 90 DAY DELETE;
```

## Practical Comparison

```sql
-- Scenario: 500-row lookup table (city -> region mapping)
-- Log is better: no index needed, zero background processes
CREATE TABLE city_region (city String, region String) ENGINE = Log;

-- Scenario: 5 billion click events
-- MergeTree is essential: index, partition, TTL
CREATE TABLE clicks (ts DateTime, user_id UInt64, url String)
ENGINE = MergeTree PARTITION BY toYYYYMM(ts) ORDER BY ts;
```

## Migration from Log to MergeTree

When a Log table outgrows its intended size:

```sql
CREATE TABLE events_v2 (
    ts      DateTime,
    type    String,
    user_id UInt64
) ENGINE = MergeTree ORDER BY (ts, user_id);

INSERT INTO events_v2 SELECT * FROM events_log;
RENAME TABLE events_log TO events_log_old, events_v2 TO events_log;
```

## Summary

Use Log family engines (TinyLog, StripeLog, Log) for small, simple tables where simplicity and minimal overhead matter more than indexing or partitioning. Use MergeTree variants for anything that requires filtering efficiency, time-based TTL, large-scale concurrency, replication, or mutations. The inflection point is roughly 1 GB of data or any query that needs a WHERE clause to be fast.
