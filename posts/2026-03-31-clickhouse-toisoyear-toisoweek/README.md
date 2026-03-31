# How to Use toISOYear() and toISOWeek() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Date Function, ISO 8601, Weekly Report, Analytics

Description: Learn how toISOYear() and toISOWeek() implement ISO 8601 week numbering for Monday-anchored weekly reports and correct year-boundary week handling.

---

ISO 8601 defines a week numbering system where weeks start on Monday and week 1 is the week containing the first Thursday of the year. This means that the last few days of December can belong to week 1 of the following year, and the first few days of January can belong to week 52 or 53 of the previous year. ClickHouse implements this with `toISOWeek(dt)`, which returns the ISO week number (1-53), and `toISOYear(dt)`, which returns the ISO year. You must use `toISOYear` alongside `toISOWeek` - never pair `toISOWeek` with `toYear` - to avoid incorrect groupings at year boundaries.

## Understanding ISO Week Numbering

```sql
-- Examine the year-boundary behavior for late December and early January
SELECT
    toDate('2024-12-28') AS dec_28,
    toYear(dec_28)       AS calendar_year,
    toISOYear(dec_28)    AS iso_year,
    toISOWeek(dec_28)    AS iso_week,

    toDate('2024-12-30') AS dec_30,
    toYear(dec_30)       AS calendar_year_30,
    toISOYear(dec_30)    AS iso_year_30,
    toISOWeek(dec_30)    AS iso_week_30,

    toDate('2024-12-31') AS dec_31,
    toISOYear(dec_31)    AS iso_year_31,
    toISOWeek(dec_31)    AS iso_week_31;
```

```text
dec_28      calendar_year  iso_year  iso_week  dec_30      calendar_year_30  iso_year_30  iso_week_30  dec_31      iso_year_31  iso_week_31
2024-12-28  2024           2024      52        2024-12-30  2024              2025         1            2024-12-31  2025         1
```

December 30 and 31, 2024 belong to ISO week 1 of ISO year 2025. Using `toYear` instead of `toISOYear` would incorrectly put them in year 2024.

## Correct GROUP BY for Weekly Reports

Always group by both `toISOYear` and `toISOWeek` to avoid mixing data from different years' week 1.

```sql
-- Weekly KPI report using correct ISO year + week grouping
SELECT
    toISOYear(event_date)  AS iso_year,
    toISOWeek(event_date)  AS iso_week,
    -- Human-readable label: year-Wweek
    concat(toString(iso_year), '-W', lpad(toString(iso_week), 2, '0')) AS week_label,
    count()                AS events,
    uniq(user_id)          AS unique_users
FROM user_events
WHERE event_date >= today() - 90
GROUP BY iso_year, iso_week
ORDER BY iso_year, iso_week;
```

## The Wrong Way: Pairing toYear With toISOWeek

This is a common mistake. Never mix `toYear` with `toISOWeek`:

```sql
-- INCORRECT: toYear and toISOWeek can disagree at year boundaries
SELECT
    toYear(event_date)    AS wrong_year,  -- says 2024 for Dec 31 2024
    toISOWeek(event_date) AS iso_week,    -- says week 1 (of 2025)
    count() AS events
FROM user_events
GROUP BY wrong_year, iso_week;
-- This merges ISO week 1 of 2025 with ISO week 1 of 2024, producing wrong totals
```

Always use `toISOYear` when using `toISOWeek`.

## Filtering by ISO Week

To query data for a specific ISO week, filter on both `toISOYear` and `toISOWeek`.

```sql
-- Query all events in ISO week 1 of 2025
SELECT
    event_id,
    event_date,
    event_type,
    user_id
FROM user_events
WHERE
    toISOYear(event_date) = 2025
    AND toISOWeek(event_date) = 1
ORDER BY event_date;
```

## Week-Over-Week Growth

Comparing a metric between consecutive ISO weeks is straightforward with window functions.

```sql
-- Week-over-week revenue growth
SELECT
    iso_year,
    iso_week,
    week_label,
    revenue,
    revenue - lagInFrame(revenue) OVER (ORDER BY iso_year, iso_week) AS wow_change,
    round(
        (revenue - lagInFrame(revenue) OVER (ORDER BY iso_year, iso_week)) * 100.0
        / nullIf(lagInFrame(revenue) OVER (ORDER BY iso_year, iso_week), 0),
        2
    ) AS wow_pct_change
FROM (
    SELECT
        toISOYear(sale_date)  AS iso_year,
        toISOWeek(sale_date)  AS iso_week,
        concat(toString(toISOYear(sale_date)), '-W', lpad(toString(toISOWeek(sale_date)), 2, '0')) AS week_label,
        sum(amount) AS revenue
    FROM sales
    WHERE sale_date >= today() - 180
    GROUP BY iso_year, iso_week
)
ORDER BY iso_year, iso_week;
```

## Computing the Monday Start of an ISO Week

`toMonday(dt)` returns the Monday that begins the ISO week containing `dt`.

```sql
-- Display the Monday start date alongside ISO year and week
SELECT
    toISOYear(event_date)  AS iso_year,
    toISOWeek(event_date)  AS iso_week,
    toMonday(event_date)   AS week_start_monday,
    count()                AS events
FROM user_events
WHERE event_date >= today() - 28
GROUP BY iso_year, iso_week, week_start_monday
ORDER BY week_start_monday;
```

## Summary

`toISOWeek(dt)` and `toISOYear(dt)` implement ISO 8601 week numbering: weeks start on Monday and week 1 contains the first Thursday of the year. Always group by `toISOYear` and `toISOWeek` together - never pair `toISOWeek` with `toYear` because they can disagree for dates in late December and early January. Use `toMonday(dt)` to get the human-readable Monday start date for a given ISO week. This combination is the correct foundation for any weekly KPI report that must align with ISO calendar conventions.
