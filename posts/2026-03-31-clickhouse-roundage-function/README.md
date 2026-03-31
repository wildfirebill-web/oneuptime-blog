# How to Use roundAge() Function in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Math Function, Demographics, Analytics

Description: Learn how roundAge() maps raw age values to standard demographic buckets in ClickHouse, with examples for age-group segmentation and marketing data analysis.

---

`roundAge()` is a convenience function in ClickHouse that maps a raw age value to a standard demographic bucket. It rounds down to the nearest value from a predefined set of age group boundaries: 0, 17, 18, 25, 35, 45, 55, and 65. These buckets correspond to commonly used demographic segments in advertising, marketing research, and product analytics. Using `roundAge()` saves you from writing a CASE expression with hardcoded boundaries every time you need age-group analysis.

## Function Signature

```text
roundAge(age)
```

Accepts any numeric type. Returns UInt8. The output is always one of the fixed boundary values: 0, 17, 18, 25, 35, 45, 55, or 65.

## Bucket Mapping

The function maps age values to the following boundaries:

```text
age  0 - 16  -> 0   (children / under 17)
age 17        -> 17  (pre-adult)
age 18 - 24  -> 18  (young adults)
age 25 - 34  -> 25  (adults)
age 35 - 44  -> 35  (mid adults)
age 45 - 54  -> 45  (mature adults)
age 55 - 64  -> 55  (pre-senior)
age 65+      -> 65  (senior)
```

## Basic Usage

Verify the bucket assignment for a range of ages.

```sql
SELECT
    age,
    roundAge(age) AS age_bucket
FROM (
    SELECT arrayJoin([5, 16, 17, 18, 24, 25, 34, 35, 44, 45, 54, 55, 64, 65, 80]) AS age
)
ORDER BY age;
```

## Setting Up Sample User Data

Create a table of user profiles with raw age values to demonstrate demographic segmentation.

```sql
CREATE TABLE user_profiles
(
    user_id     UInt64,
    username    String,
    age         UInt8,
    country     String,
    plan        String
)
ENGINE = MergeTree
ORDER BY user_id;

INSERT INTO user_profiles VALUES
(1,  'alice',   27, 'US',  'premium'),
(2,  'bob',     34, 'UK',  'free'),
(3,  'carol',   19, 'US',  'premium'),
(4,  'dave',    45, 'CA',  'free'),
(5,  'eve',     62, 'AU',  'premium'),
(6,  'frank',   16, 'US',  'free'),
(7,  'grace',   38, 'DE',  'premium'),
(8,  'heidi',   71, 'FR',  'free'),
(9,  'ivan',    22, 'US',  'premium'),
(10, 'judy',    55, 'UK',  'free');
```

## Segmenting Users by Age Group

Apply `roundAge()` to assign each user to a demographic segment, then use a CASE expression to attach a readable label.

```sql
SELECT
    user_id,
    username,
    age,
    roundAge(age) AS age_bucket,
    CASE roundAge(age)
        WHEN  0 THEN 'Under 17'
        WHEN 17 THEN 'Age 17'
        WHEN 18 THEN '18-24'
        WHEN 25 THEN '25-34'
        WHEN 35 THEN '35-44'
        WHEN 45 THEN '45-54'
        WHEN 55 THEN '55-64'
        WHEN 65 THEN '65+'
    END AS age_group
FROM user_profiles
ORDER BY age;
```

## Aggregating Users per Age Group

Count users in each age bucket to build a demographic distribution table.

```sql
SELECT
    roundAge(age)  AS age_bucket,
    count()        AS user_count,
    countIf(plan = 'premium') AS premium_count,
    round(100 * countIf(plan = 'premium') / count(), 1) AS premium_pct
FROM user_profiles
GROUP BY age_bucket
ORDER BY age_bucket;
```

## Cross-Tabulating Age Group with Country

Create a pivot table showing user distribution by age group and country.

```sql
SELECT
    roundAge(age) AS age_bucket,
    countIf(country = 'US') AS us_users,
    countIf(country = 'UK') AS uk_users,
    countIf(country = 'CA') AS ca_users,
    countIf(country = 'AU') AS au_users,
    countIf(country = 'DE') AS de_users,
    countIf(country = 'FR') AS fr_users
FROM user_profiles
GROUP BY age_bucket
ORDER BY age_bucket;
```

## Combining roundAge() with Other Aggregations

Use `roundAge()` in a GROUP BY to compute average plan upgrade rates and revenue metrics per demographic.

```sql
CREATE TABLE purchase_events
(
    user_id    UInt64,
    amount     Float64,
    ts         DateTime
)
ENGINE = MergeTree
ORDER BY (user_id, ts);

INSERT INTO purchase_events VALUES
(1, 29.99, '2024-01-15 10:00:00'),
(3, 29.99, '2024-01-16 11:00:00'),
(5, 29.99, '2024-01-17 09:00:00'),
(7, 29.99, '2024-01-18 14:00:00'),
(9, 29.99, '2024-01-19 16:00:00');
```

```sql
SELECT
    roundAge(u.age)  AS age_bucket,
    count(DISTINCT u.user_id)               AS total_users,
    count(DISTINCT p.user_id)               AS paying_users,
    round(100 * count(DISTINCT p.user_id) / count(DISTINCT u.user_id), 1) AS conversion_pct,
    round(sum(p.amount), 2)                 AS total_revenue
FROM user_profiles u
LEFT JOIN purchase_events p ON u.user_id = p.user_id
GROUP BY age_bucket
ORDER BY age_bucket;
```

## Comparing with roundDown()

`roundAge()` is equivalent to calling `roundDown(age, [0, 17, 18, 25, 35, 45, 55, 65])`. Use `roundAge()` when the standard demographic buckets match your use case; use `roundDown()` with a custom array when you need different boundaries.

```sql
SELECT
    age,
    roundAge(age)                                        AS roundage_result,
    roundDown(age, [0, 17, 18, 25, 35, 45, 55, 65])     AS rounddown_equiv,
    roundAge(age) = roundDown(age, [0, 17, 18, 25, 35, 45, 55, 65]) AS are_equal
FROM (
    SELECT arrayJoin([10, 17, 20, 30, 40, 50, 60, 70]) AS age
);
```

## Summary

`roundAge()` provides a concise way to assign raw age values to standard demographic buckets (0, 17, 18, 25, 35, 45, 55, 65) in ClickHouse, returning a UInt8. Use it directly in GROUP BY for demographic aggregation, in CASE expressions for readable group labels, and in cross-tabulations for marketing and product analytics. It is functionally equivalent to `roundDown(age, [0, 17, 18, 25, 35, 45, 55, 65])` but more readable when the standard demographic buckets match your reporting needs. For custom age groupings, use `roundDown()` with a tailored boundary array instead.
