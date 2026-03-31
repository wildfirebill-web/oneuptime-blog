# How to Use bitmapXor() and bitmapAndnot() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Bitmap, Set Operation, Analytics, Query Optimization

Description: Learn how bitmapXor() finds symmetric differences and bitmapAndnot() subtracts one bitmap from another in ClickHouse for precise audience exclusion and change-detection queries.

---

ClickHouse provides `bitmapXor(a, b)` and `bitmapAndnot(a, b)` for two additional set-theoretic bitmap operations. `bitmapXor` returns the symmetric difference: elements present in either bitmap but not in both. `bitmapAndnot` returns the asymmetric difference: elements in the first bitmap that are not in the second. Both accept `AggregateFunction(groupBitmap, UInt64)` values and return a new bitmap of the same type.

## Setting Up Sample Bitmaps

```sql
CREATE TABLE cohort_bitmaps
(
    cohort       String,
    user_bitmap  AggregateFunction(groupBitmap, UInt64)
)
ENGINE = AggregatingMergeTree()
ORDER BY cohort;
```

```sql
-- Cohort A: users 1-600, Cohort B: users 401-900
INSERT INTO cohort_bitmaps
SELECT cohort, groupBitmapState(toUInt64(user_id)) AS user_bitmap
FROM (
    SELECT 'cohort_a' AS cohort, number AS user_id FROM numbers(1, 600)
    UNION ALL
    SELECT 'cohort_b' AS cohort, number AS user_id FROM numbers(401, 500)
)
GROUP BY cohort;
```

## bitmapAndnot() - Asymmetric Difference (Subtraction)

`bitmapAndnot(a, b)` returns elements in `a` that do not appear in `b`. Think of it as `a MINUS b`.

```sql
-- Users in cohort_a but NOT in cohort_b (IDs 1-400)
SELECT
    bitmapCardinality(
        bitmapAndnot(
            (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_a'),
            (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_b')
        )
    ) AS only_in_a;
```

```text
only_in_a
400
```

```sql
-- Users in cohort_b but NOT in cohort_a (IDs 601-900)
SELECT
    bitmapCardinality(
        bitmapAndnot(
            (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_b'),
            (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_a')
        )
    ) AS only_in_b;
```

```text
only_in_b
300
```

## bitmapXor() - Symmetric Difference

`bitmapXor(a, b)` returns elements that are in exactly one of the two bitmaps - the elements unique to each side combined.

```sql
-- Users in cohort_a or cohort_b but NOT in both (IDs 1-400 and 601-900)
SELECT
    bitmapCardinality(
        bitmapXor(
            (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_a'),
            (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_b')
        )
    ) AS symmetric_diff_count;
```

```text
symmetric_diff_count
700
```

## Inspecting Symmetric Difference Results

```sql
WITH
    (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_a') AS bm_a,
    (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_b') AS bm_b
SELECT
    arraySort(bitmapToArray(bitmapXor(bm_a, bm_b))) AS xor_ids;
-- Result contains IDs 1-400 and 601-900 (elements exclusive to each cohort)
```

## Practical Use: Audience Exclusion

`bitmapAndnot` is the canonical way to exclude a suppression list from a target audience.

```sql
-- Exclude unsubscribed users from an email campaign target
WITH
    (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_a')  AS target_audience,
    (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_b')  AS unsubscribed
SELECT
    bitmapCardinality(bitmapAndnot(target_audience, unsubscribed)) AS sendable_users;
```

## Practical Use: Change Detection Between Snapshots

`bitmapXor` is ideal for finding which user IDs changed membership between two daily snapshots.

```sql
-- Compare yesterday's active users vs today's active users
WITH
    (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_a') AS yesterday_active,
    (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_b') AS today_active
SELECT
    bitmapCardinality(bitmapAndnot(yesterday_active, today_active)) AS churned_today,
    bitmapCardinality(bitmapAndnot(today_active, yesterday_active)) AS acquired_today,
    bitmapCardinality(bitmapXor(yesterday_active, today_active))    AS total_changed;
```

## Combining andnot with and/or

Build multi-step targeting logic by mixing all bitmap operators.

```sql
-- "newsletter users who are on mobile_app, but exclude paid_plan users"
-- (newsletter AND mobile_app) ANDNOT paid_plan
INSERT INTO cohort_bitmaps
SELECT cohort, groupBitmapState(toUInt64(user_id)) AS user_bitmap
FROM (
    SELECT 'newsletter' AS cohort, number AS user_id FROM numbers(1, 500)
    UNION ALL
    SELECT 'mobile_app' AS cohort, number AS user_id FROM numbers(251, 500)
    UNION ALL
    SELECT 'paid_plan'  AS cohort, number AS user_id FROM numbers(401, 400)
)
GROUP BY cohort;
```

```sql
WITH
    (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'newsletter') AS bm_nl,
    (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'mobile_app') AS bm_mob,
    (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'paid_plan')  AS bm_paid
SELECT
    bitmapCardinality(
        bitmapAndnot(
            bitmapAnd(bm_nl, bm_mob),
            bm_paid
        )
    ) AS targeted_free_mobile_newsletter_users;
```

## Verifying the Relationship Between XOR and AND/OR

The symmetric difference identity holds: `XOR(a, b) = OR(a, b) ANDNOT AND(a, b)`.

```sql
WITH
    (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_a') AS bm_a,
    (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_b') AS bm_b
SELECT
    bitmapCardinality(bitmapXor(bm_a, bm_b))                              AS xor_count,
    bitmapCardinality(bitmapAndnot(bitmapOr(bm_a, bm_b), bitmapAnd(bm_a, bm_b))) AS manual_xor_count;
-- Both columns should be equal
```

## bitmapAndnotCardinality and bitmapXorCardinality

ClickHouse also ships dedicated cardinality shortcuts that avoid materializing the result bitmap.

```sql
WITH
    (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_a') AS bm_a,
    (SELECT user_bitmap FROM cohort_bitmaps WHERE cohort = 'cohort_b') AS bm_b
SELECT
    bitmapAndnotCardinality(bm_a, bm_b) AS andnot_count,
    bitmapXorCardinality(bm_a, bm_b)    AS xor_count;
```

Use `bitmapAndnotCardinality` and `bitmapXorCardinality` when you only need the count and want to avoid the overhead of constructing a full result bitmap.

## Summary

`bitmapAndnot(a, b)` subtracts `b` from `a`, returning elements exclusive to `a`. `bitmapXor(a, b)` returns all elements exclusive to either side, forming the symmetric difference. Use `bitmapAndnot` for audience exclusion and suppression list logic, and `bitmapXor` for snapshot diffing and change detection. When only the count matters, prefer `bitmapAndnotCardinality` and `bitmapXorCardinality` to skip materializing the result bitmap.
