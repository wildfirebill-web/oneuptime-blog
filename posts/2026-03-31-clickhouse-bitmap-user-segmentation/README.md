# How to Use Bitmap Functions for User Segmentation in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Bitmap, User Segmentation, groupBitmap, bitmapAnd, bitmapOr, Roaring Bitmap

Description: Learn how to perform fast user segmentation in ClickHouse using roaring bitmap functions to compute intersections, unions, and exclusions across audience segments.

---

User segmentation - finding the overlap, union, or difference of audience sets - is a classic analytics problem. ClickHouse roaring bitmaps make these operations extremely fast even on billions of user IDs, because bitmap set operations are orders of magnitude faster than JOIN or INTERSECT on raw ID tables.

## Building a Segment Bitmap Table

The foundation is a table that stores one bitmap per segment per day:

```sql
CREATE TABLE user_segments
(
    segment_name  LowCardinality(String),
    segment_date  Date,
    user_bitmap   AggregateFunction(groupBitmap, UInt32)
)
ENGINE = AggregatingMergeTree
PARTITION BY toYYYYMM(segment_date)
ORDER BY (segment_name, segment_date);
```

Populate it from raw events using a materialized view:

```sql
CREATE MATERIALIZED VIEW user_segments_mv TO user_segments
AS
SELECT
    behavior_type   AS segment_name,
    toDate(event_time) AS segment_date,
    groupBitmapState(toUInt32(user_id)) AS user_bitmap
FROM user_events
GROUP BY segment_name, segment_date;
```

## Counting Users in a Segment

```sql
SELECT
    segment_name,
    bitmapCardinality(groupBitmapMergeState(user_bitmap)) AS user_count
FROM user_segments
WHERE segment_date = today()
GROUP BY segment_name
ORDER BY user_count DESC;
```

## Intersection - Users in Both Segments

Find users who are both "paid subscribers" AND "active this week":

```sql
SELECT bitmapCardinality(bitmapAnd(
    (SELECT groupBitmapMergeState(user_bitmap) FROM user_segments WHERE segment_name = 'paid_subscriber' AND segment_date = today()),
    (SELECT groupBitmapMergeState(user_bitmap) FROM user_segments WHERE segment_name = 'active_this_week' AND segment_date = today())
)) AS overlap_count;
```

## Union - Users in Either Segment

Count the combined reach of two campaigns:

```sql
SELECT bitmapCardinality(bitmapOr(
    (SELECT groupBitmapMergeState(user_bitmap) FROM user_segments WHERE segment_name = 'campaign_a'),
    (SELECT groupBitmapMergeState(user_bitmap) FROM user_segments WHERE segment_name = 'campaign_b')
)) AS combined_reach;
```

## Exclusion - Users in A but Not B

Find paid subscribers who have NOT used the mobile app:

```sql
SELECT bitmapCardinality(bitmapAndnot(
    (SELECT groupBitmapMergeState(user_bitmap) FROM user_segments WHERE segment_name = 'paid_subscriber'),
    (SELECT groupBitmapMergeState(user_bitmap) FROM user_segments WHERE segment_name = 'mobile_user')
)) AS desktop_only_paid;
```

## Multi-Segment Funnel

Count users who progressed through each funnel step:

```sql
WITH
    visited AS (SELECT groupBitmapMergeState(user_bitmap) FROM user_segments WHERE segment_name = 'visited_product_page'),
    added   AS (SELECT groupBitmapMergeState(user_bitmap) FROM user_segments WHERE segment_name = 'added_to_cart'),
    purchased AS (SELECT groupBitmapMergeState(user_bitmap) FROM user_segments WHERE segment_name = 'purchased')
SELECT
    bitmapCardinality(visited)                  AS step1_visited,
    bitmapCardinality(bitmapAnd(visited, added)) AS step2_added,
    bitmapCardinality(bitmapAnd(bitmapAnd(visited, added), purchased)) AS step3_purchased;
```

## Performance Advantage

Bitmap operations on a 10-million user segment typically complete in milliseconds. The equivalent JOIN on raw ID tables with the same cardinality can take seconds or minutes, making bitmaps the right tool for interactive segmentation dashboards.

## Summary

ClickHouse roaring bitmaps enable real-time user segmentation at scale. Use `groupBitmapState` and `groupBitmapMerge` to build and query segments, `bitmapAnd` for intersection, `bitmapOr` for union, and `bitmapAndnot` for exclusion. Store one bitmap per segment per time period in an `AggregatingMergeTree` table for fast incremental updates.
