# How to Use addDays(), addMonths(), addYears() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Date Function, Date Arithmetic, Shorthand, Scheduling

Description: Learn how addDays(), addMonths(), and addYears() provide concise shorthand for calendar-aware date arithmetic in ClickHouse for trial expiry and renewal queries.

---

ClickHouse provides a family of shorthand date arithmetic functions - `addDays()`, `addWeeks()`, `addMonths()`, `addQuarters()`, and `addYears()` - that wrap the more general `dateAdd()` function with a fixed unit. They produce identical results to their `dateAdd` equivalents but read more naturally in SQL, especially when the unit is clear from context. This post focuses on the three most commonly used members of the family.

## Function Signatures

```text
addDays(dt, n)       -- equivalent to dateAdd('day',   n, dt)
addWeeks(dt, n)      -- equivalent to dateAdd('week',  n, dt)
addMonths(dt, n)     -- equivalent to dateAdd('month', n, dt)
addQuarters(dt, n)   -- equivalent to dateAdd('quarter', n, dt)
addYears(dt, n)      -- equivalent to dateAdd('year',  n, dt)

addHours(dt, n)      -- equivalent to dateAdd('hour',   n, dt)
addMinutes(dt, n)    -- equivalent to dateAdd('minute', n, dt)
addSeconds(dt, n)    -- equivalent to dateAdd('second', n, dt)
```

All functions accept a `Date`, `DateTime`, or `DateTime64` and return the same type. `n` is a signed integer - negative values subtract.

## Basic Usage

The examples below add a range of intervals to a single base date.

```sql
SELECT
    toDate('2026-03-31')              AS base,
    addDays(toDate('2026-03-31'), 7)  AS plus_1_week,
    addDays(toDate('2026-03-31'), 30) AS plus_30_days,
    addMonths(toDate('2026-03-31'), 1) AS plus_1_month,
    addMonths(toDate('2026-03-31'), 6) AS plus_6_months,
    addYears(toDate('2026-03-31'), 1)  AS plus_1_year,
    addYears(toDate('2026-03-31'), -1) AS minus_1_year;
```

## Trial Expiry Calculation

A 14-day free trial expires exactly 14 days after signup. `addDays` is the most readable way to express this.

```sql
SELECT
    user_id,
    signup_date,
    addDays(signup_date, 14)   AS trial_expiry,
    dateDiff('day', today(), addDays(signup_date, 14)) AS days_remaining
FROM users
WHERE plan = 'trial'
  AND addDays(signup_date, 14) >= today()
ORDER BY trial_expiry ASC;
```

## Monthly and Annual Renewal Dates

Subscription renewal dates should be computed with `addMonths` or `addYears` rather than adding a fixed number of days. Calendar-aware addition means a monthly subscription signed on January 31 renews on February 28 rather than an invalid date.

```sql
SELECT
    user_id,
    subscription_start,
    addMonths(subscription_start, billing_cycle_months) AS next_renewal,
    addYears(subscription_start, 1)                     AS annual_equivalent
FROM subscriptions
WHERE status = 'active'
ORDER BY next_renewal ASC;
```

## Anniversary and Milestone Calculations

Finding the annual anniversary of any event is straightforward with `addYears`.

```sql
SELECT
    employee_id,
    name,
    hire_date,
    addYears(hire_date, 1)  AS first_anniversary,
    addYears(hire_date, 5)  AS five_year_milestone,
    dateDiff('year', hire_date, today()) AS years_of_service
FROM employees
ORDER BY years_of_service DESC;
```

## Generating a Renewal Calendar

You can generate the full sequence of future renewal dates by combining `addMonths` with `arrayJoin` and `range()`.

```sql
SELECT
    user_id,
    subscription_start,
    addMonths(subscription_start, number + 1) AS renewal_date
FROM subscriptions
CROSS JOIN (
    SELECT arrayJoin(range(0, 12)) AS number
) AS months
WHERE status = 'active'
  AND renewal_date >= today()
ORDER BY user_id, renewal_date ASC;
```

## Combining addDays with toStartOfDay

When you need "the start of the day N days from now", combine `addDays` with `toStartOfDay` to produce a clean midnight boundary.

```sql
SELECT
    toStartOfDay(addDays(now(), -7))  AS seven_days_ago_midnight,
    toStartOfDay(addDays(now(),  0))  AS today_midnight,
    toStartOfDay(addDays(now(),  1))  AS tomorrow_midnight;
```

## Negative n for Lookback Windows

All shorthand functions accept negative `n`, making them equivalent to their `subtract*` counterparts. Both styles are valid; choose based on readability.

```sql
-- These two queries are equivalent:
SELECT count() FROM events WHERE event_time >= addDays(now(), -30);
SELECT count() FROM events WHERE event_time >= subtractDays(now(), 30);
```

## Summary

`addDays()`, `addMonths()`, and `addYears()` are concise, readable shorthand for the most common date shifts in ClickHouse. They are fully calendar-aware, return the same type as their input, and accept negative values for backward shifts. Use them over raw `dateAdd()` calls when the unit is obvious from context, and over manual second/day arithmetic whenever calendar semantics matter (month lengths, leap years, DST).
