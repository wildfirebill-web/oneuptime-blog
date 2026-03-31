# How to Handle Sparse Data Patterns in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Sparse Data, Nullable, Map, Schema Design, Storage

Description: Learn how to efficiently store and query sparse data in ClickHouse using Nullable columns, Map types, and vertical partitioning to minimize storage overhead.

---

## What Is Sparse Data?

Sparse data means most column values are NULL or zero for a given row. Examples include:
- User attribute tables where each user has only a few of hundreds of possible properties
- Event tables with optional fields that apply to only certain event types
- IoT sensor data where not all sensors report every metric

## Option 1 - Nullable Columns

The most straightforward approach is marking optional columns as Nullable:

```sql
CREATE TABLE sensor_readings (
    sensor_id  UInt64,
    ts         DateTime,
    temperature Nullable(Float32),
    humidity    Nullable(Float32),
    pressure    Nullable(Float32),
    co2         Nullable(Float32)
) ENGINE = MergeTree()
ORDER BY (sensor_id, ts);
```

Drawback: Nullable adds a separate null-map bitmap file per column, increasing storage and read overhead. Avoid Nullable on high-cardinality columns queried frequently.

## Option 2 - Map Columns for Dynamic Attributes

When the set of attributes is not known ahead of time, use `Map`:

```sql
CREATE TABLE user_properties (
    user_id    UInt64,
    ts         DateTime,
    props      Map(LowCardinality(String), String)
) ENGINE = MergeTree()
ORDER BY (user_id, ts);
```

Insert data:

```sql
INSERT INTO user_properties VALUES
(1, now(), {'plan': 'pro', 'country': 'US'}),
(2, now(), {'plan': 'free'});
```

Query a specific key:

```sql
SELECT user_id, props['plan'] AS plan
FROM user_properties
WHERE props['country'] = 'US';
```

## Option 3 - Separate Tables per Attribute Class

Normalize sparse attributes into a key-value child table:

```sql
CREATE TABLE user_attrs (
    user_id   UInt64,
    attr_name LowCardinality(String),
    attr_val  String
) ENGINE = MergeTree()
ORDER BY (user_id, attr_name);
```

Pivot at query time with `groupArray` or conditional aggregation:

```sql
SELECT
    user_id,
    maxIf(attr_val, attr_name = 'plan')    AS plan,
    maxIf(attr_val, attr_name = 'country') AS country
FROM user_attrs
GROUP BY user_id;
```

## Option 4 - Default Values Instead of NULL

For numeric sparse data, use a sentinel default instead of Nullable:

```sql
CREATE TABLE metrics (
    host    LowCardinality(String),
    ts      DateTime,
    cpu     Float32 DEFAULT 0,
    mem     Float32 DEFAULT 0,
    disk_io Float32 DEFAULT 0
) ENGINE = MergeTree()
ORDER BY (host, ts);
```

This avoids null-map overhead and compresses better with delta or Gorilla codecs.

## Compression Tips for Sparse Data

Apply ZSTD compression to Map or String columns holding sparse data:

```sql
ALTER TABLE user_properties MODIFY COLUMN props
    Map(LowCardinality(String), String) CODEC(ZSTD(3));
```

## Summary

ClickHouse handles sparse data best through Map columns for dynamic attributes, default values for numeric fields, and separate child tables for normalized attribute storage. Avoid over-using Nullable when default sentinel values are semantically acceptable, as this reduces storage overhead and improves query performance.
