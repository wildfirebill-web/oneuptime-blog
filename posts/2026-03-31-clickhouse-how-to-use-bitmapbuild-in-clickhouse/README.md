# How to Use bitmapBuild() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Bitmap, bitmapBuild, Aggregation, Sql

Description: Learn how to use bitmapBuild() in ClickHouse to construct compressed bitmap objects from arrays of unsigned integers for efficient set operations.

---

## Overview

ClickHouse supports compressed bitmaps (Roaring Bitmaps) as a data type through the `AggregateFunction(groupBitmap, UInt*)` type. `bitmapBuild()` constructs a bitmap from an array of unsigned integers, enabling efficient union, intersection, and cardinality operations.

## Creating a Bitmap with bitmapBuild()

```sql
SELECT bitmapBuild([1, 2, 3, 4, 5]) AS bm
```

The input must be an `Array(UInt*)` type (UInt8, UInt16, UInt32, or UInt64).

## Checking Bitmap Cardinality

```sql
SELECT bitmapCardinality(bitmapBuild([1, 2, 3, 4, 5])) AS cardinality
-- 5

SELECT bitmapCardinality(bitmapBuild([1, 1, 2, 2, 3])) AS cardinality
-- 3 (bitmaps deduplicate automatically)
```

## Storing Bitmaps in Tables

Bitmaps are typically stored via the `groupBitmapState` aggregate function:

```sql
CREATE TABLE user_activity_bitmap
(
    date        Date,
    feature     LowCardinality(String),
    active_users AggregateFunction(groupBitmap, UInt64)
)
ENGINE = AggregatingMergeTree()
ORDER BY (date, feature);

INSERT INTO user_activity_bitmap
SELECT
    toDate(event_time) AS date,
    feature_name       AS feature,
    groupBitmapState(user_id) AS active_users
FROM events
GROUP BY date, feature
```

## Using bitmapBuild in SELECT for Testing

Build test bitmaps inline:

```sql
SELECT
    bitmapCardinality(bitmapBuild([100, 200, 300, 400, 500])) AS size,
    bitmapContains(bitmapBuild([100, 200, 300]), toUInt32(200)) AS has_200
```

## Bitmap Union

Combine bitmaps from multiple rows:

```sql
SELECT
    feature,
    bitmapCardinality(groupBitmapMerge(active_users)) AS total_unique_users
FROM user_activity_bitmap
WHERE date BETWEEN '2024-01-01' AND '2024-01-07'
GROUP BY feature
```

## Bitmap Intersection

Find users active on both feature A and feature B:

```sql
SELECT bitmapCardinality(
    bitmapAnd(
        bitmapBuild([1, 2, 3, 4, 5]),
        bitmapBuild([3, 4, 5, 6, 7])
    )
) AS intersection_count
-- 3 (elements 3, 4, 5)
```

## Bitmap Difference

Find users in set A but not in set B:

```sql
SELECT bitmapToArray(
    bitmapAndnot(
        bitmapBuild([1, 2, 3, 4, 5]),
        bitmapBuild([3, 4, 5, 6, 7])
    )
) AS a_minus_b
-- [1, 2]
```

## Practical Use Case - DAU/WAU/MAU Funnels

Build daily active user bitmaps and compute 7-day rolling unique users:

```sql
SELECT
    date,
    groupBitmapMerge(active_users) AS week_bitmap,
    bitmapCardinality(groupBitmapMerge(active_users)) AS wau
FROM user_activity_bitmap
WHERE date BETWEEN today() - 7 AND today()
GROUP BY date
```

## Converting Back to Array

```sql
SELECT bitmapToArray(bitmapBuild([5, 3, 1, 4, 2])) AS sorted_array
-- [1, 2, 3, 4, 5] (bitmaps maintain sorted order)
```

## Summary

`bitmapBuild()` constructs a Roaring Bitmap from an array of unsigned integers in ClickHouse, with automatic deduplication. Bitmaps excel at computing DAU/WAU/MAU, funnel intersection sizes, and set operations at scale. Store pre-aggregated bitmaps in `AggregatingMergeTree` tables using `groupBitmapState` for maximum performance.
