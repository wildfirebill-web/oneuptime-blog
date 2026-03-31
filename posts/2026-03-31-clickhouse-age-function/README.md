# How to Use age() Function in ClickHouse for Duration Calculation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Date Function, Duration Calculation, Analytics, Time Series

Description: Learn how age() computes calendrical differences between dates accounting for variable-length months and years, unlike dateDiff which counts boundary crossings.

---

ClickHouse's `age('unit', start, end)` function computes the calendrical difference between two dates in the specified unit, correctly accounting for the variable lengths of months and years. This is distinct from `dateDiff`, which simply counts how many unit boundaries are crossed between two timestamps. For example, `dateDiff('year', '2023-03-01', '2024-02-28')` returns `1` because it crosses the year boundary, but `age('year', '2023-03-01', '2024-02-28')` returns `0` because a full calendar year has not yet elapsed. This makes `age` the right choice for calculating user ages from birthdates, subscription durations, and accurate anniversary detection.

## Understanding the Difference Between age and dateDiff

```sql
-- Illustrate the difference between age and dateDiff
SELECT
    toDate('2023-03-01') AS start_date,
    toDate('2024-02-28') AS end_date,
    dateDiff('year', start_date, end_date)  AS datediff_years,
    age('year', start_date, end_date)       AS age_years;
```

```text
start_date  end_date    datediff_years  age_years
2023-03-01  2024-02-28  1               0
```

`dateDiff` sees that the year changed (2023 to 2024) so it returns 1. `age` verifies that a full year has not completed - March 1, 2024 has not yet arrived - so it returns 0.

## Calculating User Age From Birthdate

The most intuitive use case is computing a person's current age in years.

```sql
-- Compute the current age of each user in years
SELECT
    user_id,
    birth_date,
    age('year', birth_date, today()) AS age_years
FROM users
ORDER BY age_years DESC
LIMIT 10;
```

## Age in Multiple Units at Once

You can compute the age at different granularities simultaneously to build human-friendly duration strings.

```sql
-- Show age in years, months, and days for each user
SELECT
    user_id,
    birth_date,
    age('year',  birth_date, today()) AS years,
    age('month', birth_date, today()) AS total_months,
    age('day',   birth_date, today()) AS total_days
FROM users
LIMIT 10;
```

## Subscription Duration in Months

For SaaS products, computing how many complete months a subscription has been active is a natural fit for `age`.

```sql
-- Find customers who have been subscribed for at least 12 complete months
SELECT
    customer_id,
    plan,
    subscription_start,
    age('month', subscription_start, today()) AS complete_months_active
FROM subscriptions
WHERE
    status = 'active'
    AND age('month', subscription_start, today()) >= 12
ORDER BY complete_months_active DESC;
```

## Detecting Anniversaries

Because `age` operates calendrically, you can check whether today is an anniversary of an event by verifying that the age in days modulo 365 lands near zero - but a more accurate approach uses year-level `age`.

```sql
-- Find users whose account anniversary is today or in the next 7 days
SELECT
    user_id,
    created_at,
    age('year', created_at, today()) AS years_as_customer
FROM users
WHERE
    -- anniversary falls between today and 7 days from now
    toDayOfYear(
        toDate(created_at) + toIntervalYear(
            age('year', created_at, today()) + 1
        )
    ) BETWEEN toDayOfYear(today()) AND toDayOfYear(today() + 7);
```

## Accurate Age Bucketing

When grouping users into age cohorts, `age` gives correct bucket assignments near year boundaries where `dateDiff` would prematurely advance the bucket.

```sql
-- Group users by age bracket using age()
SELECT
    multiIf(
        age('year', birth_date, today()) < 18,  'Under 18',
        age('year', birth_date, today()) < 25,  '18-24',
        age('year', birth_date, today()) < 35,  '25-34',
        age('year', birth_date, today()) < 45,  '35-44',
        age('year', birth_date, today()) < 55,  '45-54',
        '55+'
    ) AS age_bracket,
    count() AS user_count
FROM users
WHERE birth_date IS NOT NULL
GROUP BY age_bracket
ORDER BY age_bracket;
```

## Tenure Distribution for HR Analytics

For workforce analytics, `age` in months from hire date gives precise tenure calculations regardless of how many days are in the intervening months.

```sql
-- Compute tenure distribution for active employees
SELECT
    floor(age('month', hire_date, today()) / 12) AS tenure_years,
    count() AS headcount,
    avg(salary) AS avg_salary
FROM employees
WHERE status = 'active'
GROUP BY tenure_years
ORDER BY tenure_years;
```

## Summary

`age('unit', start, end)` measures the complete calendrical units elapsed between two dates, correctly handling months of varying lengths and leap years. This makes it superior to `dateDiff` for user age calculations, subscription tenure, anniversary detection, and any scenario where crossing a boundary (such as a year rollover) does not by itself constitute a full elapsed period. Use `age` whenever the human-calendar meaning of "a year" or "a month" matters more than a raw count of boundary crossings.
