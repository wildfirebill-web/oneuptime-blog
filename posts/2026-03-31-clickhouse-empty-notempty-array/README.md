# How to Use empty() and notEmpty() for Arrays in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Array Function, empty, notEmpty, Filtering

Description: Learn how empty() and notEmpty() check whether ClickHouse arrays have zero or more elements, enabling clean filtering of rows with missing or present array data.

---

Before processing an array column, it is often necessary to first check whether it has any elements at all. Using array indexing on an empty array, or calling `arrayMin` on one, can return unexpected default values. `empty` and `notEmpty` provide clear, readable null-like checks for array columns, integrating naturally into `WHERE`, `HAVING`, and conditional expressions.

## Function Signatures

```text
empty(arr)    -> UInt8  -- returns 1 if array has 0 elements, else 0
notEmpty(arr) -> UInt8  -- returns 1 if array has 1 or more elements, else 0
```

These functions also work on strings: `empty('')` returns 1. For arrays, they check `length(arr) = 0`.

## Basic Usage

```sql
-- Check if array is empty
SELECT empty([]) AS is_empty;          -- 1
SELECT empty([1, 2, 3]) AS is_empty;   -- 0

-- Check if array has elements
SELECT notEmpty([]) AS has_data;       -- 0
SELECT notEmpty([1, 2, 3]) AS has_data; -- 1

-- They are logical complements
SELECT
    empty([1, 2]) AS emp,
    notEmpty([1, 2]) AS not_emp;
-- emp: 0, not_emp: 1

-- Works with any element type
SELECT empty(['']) AS empty_str_array;  -- 0 (array has one element, the empty string)
SELECT empty([]) AS truly_empty;        -- 1
```

## Filtering Rows with Empty Array Columns

A common need is to exclude rows where an array column has no values - for example, users with no tags, events with no associated errors, or sessions with no page views:

```sql
CREATE TABLE user_profiles
(
    user_id UInt32,
    interests Array(String),
    purchased_product_ids Array(UInt32)
) ENGINE = Memory;

INSERT INTO user_profiles VALUES
    (1, ['sports', 'technology', 'travel'], [101, 203, 305]),
    (2, ['cooking'], []),
    (3, [], [101, 102]),
    (4, [], []);

-- Only users who have at least one interest
SELECT user_id, interests
FROM user_profiles
WHERE notEmpty(interests);
-- user 1, user 2

-- Only users with no purchases
SELECT user_id
FROM user_profiles
WHERE empty(purchased_product_ids);
-- user 2, user 4

-- Users with interests but no purchases
SELECT user_id, interests
FROM user_profiles
WHERE notEmpty(interests) AND empty(purchased_product_ids);
-- user 2
```

## Using in SELECT for Conditional Logic

`empty` and `notEmpty` return 0 or 1, which means they work directly in arithmetic and conditional expressions:

```sql
SELECT
    user_id,
    notEmpty(interests)            AS has_interests,
    notEmpty(purchased_product_ids) AS has_purchases,
    -- Simple score: 1 point for interests, 1 point for purchases
    notEmpty(interests) + notEmpty(purchased_product_ids) AS engagement_score
FROM user_profiles;
-- user 1: has_interests=1, has_purchases=1, score=2
-- user 2: has_interests=1, has_purchases=0, score=1
-- user 3: has_interests=0, has_purchases=1, score=1
-- user 4: has_interests=0, has_purchases=0, score=0
```

## Avoiding Default Value Traps

Accessing an element of an empty array in ClickHouse returns a default zero-like value rather than an error. Using `notEmpty` guards against silently processing these defaults:

```sql
-- Without guard: returns 0 for users with no purchases (misleading)
SELECT user_id, purchased_product_ids[1] AS first_purchase
FROM user_profiles;
-- user 3: 101 (correct)
-- user 4: 0   (misleading default, not a real product id)

-- With guard: only return first purchase for users who have one
SELECT
    user_id,
    if(notEmpty(purchased_product_ids), purchased_product_ids[1], NULL) AS first_purchase
FROM user_profiles;
-- user 3: 101
-- user 4: NULL (correct - no purchase)
```

## Counting Empty vs Non-Empty Rows

Use `empty` and `notEmpty` in aggregations to count how many rows have or lack array data:

```sql
SELECT
    countIf(notEmpty(interests))              AS users_with_interests,
    countIf(empty(interests))                 AS users_without_interests,
    countIf(notEmpty(purchased_product_ids))  AS users_with_purchases,
    countIf(empty(purchased_product_ids))     AS users_without_purchases
FROM user_profiles;
-- users_with_interests: 2
-- users_without_interests: 2
-- users_with_purchases: 2
-- users_without_purchases: 2
```

## Validating Array Data Quality

In a data quality check, identify rows where required array columns are empty:

```sql
SELECT
    user_id,
    empty(interests) AS missing_interests,
    empty(purchased_product_ids) AS missing_purchases,
    -- Flag rows with any missing required arrays
    (empty(interests) OR empty(purchased_product_ids)) AS incomplete_profile
FROM user_profiles;
```

## Using in HAVING for Group-Level Checks

After grouping, use `notEmpty` on aggregated arrays to filter out groups that produced no elements:

```sql
CREATE TABLE events
(
    session_id UInt32,
    event_type String,
    page String
) ENGINE = Memory;

INSERT INTO events VALUES
    (1, 'page_view', 'home'),
    (1, 'page_view', 'about'),
    (1, 'click', 'home'),
    (2, 'page_view', 'home'),
    (3, 'click', 'pricing');

-- Sessions that had at least one page_view
SELECT
    session_id,
    groupArray(if(event_type = 'page_view', page, NULL)) AS viewed_pages
FROM events
GROUP BY session_id
HAVING notEmpty(arrayFilter(x -> x IS NOT NULL, viewed_pages));
```

## Difference Between empty() and length() = 0

Both approaches check for an empty array, but `empty` is more readable and intended for this purpose:

```sql
-- These are equivalent but empty() is preferred
SELECT empty([1, 2, 3])     AS using_empty;   -- 0
SELECT length([1, 2, 3]) = 0 AS using_length;  -- 0

-- notEmpty is cleaner than length() > 0
SELECT notEmpty([1, 2, 3])   AS using_notempty; -- 1
SELECT length([1, 2, 3]) > 0 AS using_length;   -- 1
```

## Summary

`empty` and `notEmpty` are the idiomatic ClickHouse way to check whether an array has zero or more elements. Use `empty` to filter out rows with no array data and `notEmpty` to ensure you only process rows with actual values. They prevent silent default-value traps from array indexing on empty arrays, work cleanly in `WHERE`, `HAVING`, `countIf`, and conditional expressions, and are more readable than the equivalent `length(arr) = 0` comparison.
