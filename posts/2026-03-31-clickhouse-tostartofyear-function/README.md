# How to Use toStartOfYear() in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Date Function, Analytics, Aggregation, Annual

Description: Learn how to use toStartOfYear() in ClickHouse for annual aggregations, year-over-year analysis, and computing days elapsed since the start of the year.

---

`toStartOfYear()` is a straightforward but powerful function in ClickHouse that snaps any date or timestamp to January 1st of the same year. It is the foundation of year-level bucketing, year-to-date filtering, and year-over-year comparisons.

This post covers how `toStartOfYear()` works, when to use it over `toYear()`, and shows a range of practical SQL patterns for annual reporting.

## toStartOfYear() - How It Works

`toStartOfYear(dt)` accepts a `Date` or `DateTime` value and returns a `Date` representing January 1st of the year that contains the input. The time component is dropped regardless of the input type.

```sql
SELECT
    toDate('2026-03-31')                    AS input_date,
    toStartOfYear(toDate('2026-03-31'))     AS year_start;
```

```text
input_date   year_start
------------ -----------
2026-03-31   2026-01-01
```

Every date in 2026 - from January 1st through December 31st - returns `2026-01-01`.

## toStartOfYear() vs. toYear()

Both functions are used for yearly grouping, but they return different types:

```text
toYear(dt)         -> UInt16  (e.g., 2026)
toStartOfYear(dt)  -> Date    (e.g., 2026-01-01)
```

Use `toYear()` when you want a compact integer label. Use `toStartOfYear()` when you need a `Date` value for range arithmetic (adding intervals, comparing to other date functions, or joining to a date dimension table).

```sql
-- Both produce yearly groupings; the first returns an integer, the second a Date
SELECT toYear(sale_date) AS year, sum(revenue) FROM sales GROUP BY year;

SELECT toStartOfYear(sale_date) AS year_start, sum(revenue) FROM sales GROUP BY year_start;
```

## Annual Aggregations

Group any metric by year using `toStartOfYear()` as the bucket key:

```sql
SELECT
    toStartOfYear(sale_date)  AS year_start,
    count()                   AS orders,
    sum(revenue)              AS annual_revenue,
    avg(revenue)              AS avg_order_value,
    uniq(customer_id)         AS unique_customers
FROM sales
GROUP BY year_start
ORDER BY year_start;
```

## Year-to-Date Filtering

Filter for data from the start of the current year through today:

```sql
SELECT
    count()        AS ytd_orders,
    sum(revenue)   AS ytd_revenue
FROM sales
WHERE sale_date >= toStartOfYear(today());
```

For a prior-year-to-date comparison at the same point in the calendar:

```sql
SELECT
    toYear(sale_date)  AS year,
    sum(revenue)       AS ytd_revenue
FROM sales
WHERE
    -- Same ordinal day range in both years
    toDayOfYear(sale_date) <= toDayOfYear(today())
    AND toYear(sale_date) IN (toYear(today()), toYear(today()) - 1)
GROUP BY year
ORDER BY year;
```

## Year-over-Year Revenue Comparison

Use a window function to calculate year-over-year growth:

```sql
SELECT
    year_start,
    annual_revenue,
    lagInFrame(annual_revenue) OVER (ORDER BY year_start) AS prev_year_revenue,
    round(
        (annual_revenue - lagInFrame(annual_revenue) OVER (ORDER BY year_start))
        / lagInFrame(annual_revenue) OVER (ORDER BY year_start) * 100,
        2
    ) AS yoy_growth_pct
FROM (
    SELECT
        toStartOfYear(sale_date)  AS year_start,
        sum(revenue)              AS annual_revenue
    FROM sales
    GROUP BY year_start
)
ORDER BY year_start;
```

## Computing Days Elapsed Since Start of Year

Subtract the year start from today to get the number of days elapsed so far in the current year:

```sql
SELECT
    today()                          AS today,
    toStartOfYear(today())           AS year_start,
    today() - toStartOfYear(today()) AS days_elapsed_in_year;
```

```text
today        year_start   days_elapsed_in_year
------------ ------------ --------------------
2026-03-31   2026-01-01   89
```

This is useful for normalizing metrics (revenue per day so far) or for pro-rating annual targets.

## Annualized Revenue Projection

Using days elapsed, project the full-year revenue from year-to-date actuals:

```sql
SELECT
    sum(revenue)                            AS ytd_revenue,
    today() - toStartOfYear(today())        AS days_elapsed,
    round(
        sum(revenue)
        / (today() - toStartOfYear(today()))
        * 365,
        2
    )                                       AS projected_annual_revenue
FROM sales
WHERE sale_date >= toStartOfYear(today());
```

## Multi-Year Trend with toStartOfYear() as a Join Key

If you have a separate targets or budgets table keyed by `Date`, `toStartOfYear()` makes it easy to join actuals to annual targets:

```sql
SELECT
    a.year_start,
    a.annual_revenue,
    t.target_revenue,
    round(a.annual_revenue / t.target_revenue * 100, 1) AS pct_of_target
FROM (
    SELECT
        toStartOfYear(sale_date)  AS year_start,
        sum(revenue)              AS annual_revenue
    FROM sales
    GROUP BY year_start
) AS a
LEFT JOIN annual_targets AS t ON a.year_start = t.year_start
ORDER BY a.year_start;
```

## Filtering Within a Specific Year

Use `toStartOfYear()` to define both the lower and upper bounds of a year range dynamically:

```sql
-- All data from calendar year 2025
SELECT *
FROM events
WHERE event_date >= toStartOfYear(toDate('2025-06-01'))
  AND event_date <  toStartOfYear(toDate('2026-06-01'));
```

Passing any date in the target year to `toStartOfYear()` returns January 1st of that year, making this pattern robust regardless of which date you pass in.

## Summary

`toStartOfYear()` returns the `Date` value `YYYY-01-01` for any input and is the cleanest way to bucket data into annual periods in ClickHouse. Use it for GROUP BY year buckets when you need a `Date` type (rather than the integer returned by `toYear()`), for year-to-date range filters, for days-elapsed calculations, and for joining actuals to annual target tables. Combined with window functions it powers straightforward year-over-year growth queries.
