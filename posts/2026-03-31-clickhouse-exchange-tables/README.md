# How to Use EXCHANGE TABLES in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, DDL, EXCHANGE TABLES, Atomic Swap

Description: Learn how to atomically swap two tables in ClickHouse using EXCHANGE TABLES, enabling zero-downtime blue-green data refreshes without a temporary name.

---

EXCHANGE TABLES swaps the names of two existing tables in a single atomic metadata operation. Unlike a multi-step RENAME TABLE pattern, EXCHANGE TABLES guarantees that at no point in time does either name disappear - the swap is instantaneous at the metadata level. This makes it the preferred mechanism for blue-green data refreshes in ClickHouse production environments.

## Basic EXCHANGE TABLES Syntax

```sql
EXCHANGE TABLES table_a AND table_b;
```

With database qualifiers:

```sql
EXCHANGE TABLES analytics.events AND analytics.events_new;
```

After this statement, `analytics.events` points to what was previously `analytics.events_new`, and vice versa. All data, parts, and storage remain in place - only the metadata names are swapped.

## How EXCHANGE TABLES Differs from RENAME TABLE

With RENAME TABLE you need at least two steps and a temporary name to perform a swap without a gap:

```sql
-- Multi-step rename approach (has atomicity concerns)
RENAME TABLE analytics.events     TO analytics.events_tmp;
RENAME TABLE analytics.events_new TO analytics.events;
RENAME TABLE analytics.events_tmp TO analytics.events_old;
```

With EXCHANGE TABLES there is no intermediate state:

```sql
-- Single atomic swap - no gap, no temporary name needed
EXCHANGE TABLES analytics.events AND analytics.events_new;
```

The two tables must exist in the same database (or you must fully qualify both). They do not need to have the same schema, though in practice they almost always do for this pattern to be useful.

## Blue-Green Data Refresh Pattern

The canonical use case is refreshing a large aggregated or materialized table without any downtime for readers:

```sql
-- Step 1: create or truncate the shadow table with the same schema
CREATE TABLE IF NOT EXISTS analytics.events_shadow AS analytics.events;
TRUNCATE TABLE analytics.events_shadow;

-- Step 2: populate the shadow table with fresh data
INSERT INTO analytics.events_shadow
SELECT
    event_id,
    event_type,
    user_id,
    toStartOfHour(created_at) AS created_at,
    count()                   AS event_count
FROM raw.events_raw
WHERE created_at >= today() - 1
GROUP BY event_id, event_type, user_id, created_at;

-- Step 3: atomically swap shadow into production
EXCHANGE TABLES analytics.events AND analytics.events_shadow;

-- Step 4: events_shadow now holds the old data; drop or keep it
DROP TABLE IF EXISTS analytics.events_shadow;
```

Readers querying `analytics.events` see either the old data or the new data - never a partial or empty state.

## ON CLUSTER

```sql
EXCHANGE TABLES analytics.events AND analytics.events_new ON CLUSTER my_cluster;
```

The exchange is propagated to all nodes in the named cluster.

## Verifying the Swap

```sql
-- Check row counts before and after to confirm the swap occurred
SELECT count() FROM analytics.events;
SELECT count() FROM analytics.events_shadow;
```

```sql
-- Inspect the modification time in system tables
SELECT
    name,
    engine,
    total_rows,
    metadata_modification_time
FROM system.tables
WHERE database = 'analytics'
  AND name IN ('events', 'events_shadow');
```

## Practical Example - Dictionary Refresh

EXCHANGE TABLES is also useful for refreshing lookup tables that feed dictionaries:

```sql
-- Rebuild the lookup table in the background
CREATE TABLE IF NOT EXISTS dim.products_new AS dim.products;
TRUNCATE TABLE dim.products_new;

INSERT INTO dim.products_new SELECT * FROM raw.products_raw;

-- Swap into production and reload the dictionary
EXCHANGE TABLES dim.products AND dim.products_new;
SYSTEM RELOAD DICTIONARY dim.products_dict;

DROP TABLE IF EXISTS dim.products_new;
```

## Summary

EXCHANGE TABLES provides a single-statement atomic name swap between two existing ClickHouse tables, eliminating the brief gap inherent in multi-step RENAME TABLE approaches. It is the cleanest way to implement a blue-green data refresh pattern and ensures readers always see a complete, consistent table regardless of when they execute their queries.
