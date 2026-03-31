# How to Use bitmapHasAny() and bitmapHasAll() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Bitmap, bitmapHasAny, bitmapHasAll, Set Operation, Analytics

Description: Learn how to use bitmapHasAny() and bitmapHasAll() in ClickHouse to check bitmap membership for any or all elements in set intersection queries.

---

ClickHouse provides `bitmapHasAny()` and `bitmapHasAll()` for membership testing on Roaring Bitmaps. These functions let you check whether a bitmap contains at least one element from a set (`HasAny`) or every element from a set (`HasAll`).

## Function Signatures

```sql
bitmapHasAny(bitmap, bitmap) -> UInt8
bitmapHasAll(bitmap, bitmap) -> UInt8
```

Both functions return 1 (true) or 0 (false). The second argument is the set of values to test against the first bitmap.

## Basic Usage

```sql
SELECT
    bitmapHasAny(bitmapBuild([1, 2, 3, 4, 5]), bitmapBuild([3, 7, 9])) AS has_any,
    bitmapHasAll(bitmapBuild([1, 2, 3, 4, 5]), bitmapBuild([1, 2, 3])) AS has_all
```

```text
has_any  has_all
-------  -------
1        1
```

`has_any` is 1 because 3 appears in both bitmaps. `has_all` is 1 because 1, 2, and 3 are all in the first bitmap.

## Audience Overlap Detection

Check if a user belongs to any of several target cohorts:

```sql
SELECT
    user_id,
    bitmapHasAny(user_tags, campaign_tags) AS is_eligible
FROM user_profiles
CROSS JOIN (
    SELECT bitmapBuild([101, 205, 310]) AS campaign_tags
) AS campaign
WHERE is_eligible = 1
```

## Full Inclusion Check

Use `bitmapHasAll()` to find users that have acquired every feature in a feature set:

```sql
SELECT
    user_id,
    bitmapHasAll(feature_flags, bitmapBuild([1, 2, 5])) AS has_full_access
FROM user_features
WHERE has_full_access = 1
```

This is equivalent to checking that a user has features 1, 2, AND 5 all enabled.

## Filtering Campaign Targets

```sql
SELECT
    segment_name,
    bitmapCardinality(segment_bitmap) AS segment_size,
    bitmapHasAny(segment_bitmap, bitmapBuild([1001, 1002, 1003])) AS contains_vip_users
FROM audience_segments
ORDER BY segment_name
```

## Combining with groupBitmap()

You can build bitmaps on the fly with `groupBitmap()` and test membership:

```sql
WITH
    (SELECT groupBitmap(user_id) FROM churned_users) AS churned_bitmap
SELECT
    campaign_id,
    bitmapHasAny(target_users, churned_bitmap) AS targets_churned_users
FROM campaigns
```

This quickly flags any campaign that targets previously churned users.

## Performance Comparison vs IN

For large sets (thousands of IDs), bitmap membership is faster than `IN (...)` because:
- Bitmaps are compressed in memory
- Intersection is computed with bitwise AND in O(n) time
- No need to scan a list for each comparison

```sql
-- Slower for large sets:
WHERE user_id IN (101, 205, 310, ...)

-- Faster with bitmaps:
WHERE bitmapHasAny(user_bitmap, target_bitmap) = 1
```

## Summary

`bitmapHasAny()` and `bitmapHasAll()` are efficient membership-testing functions for Roaring Bitmaps in ClickHouse. Use `bitmapHasAny()` for OR-style membership checks and `bitmapHasAll()` for AND-style inclusion checks. They are especially useful in user segmentation, feature flag evaluation, and campaign targeting pipelines.
