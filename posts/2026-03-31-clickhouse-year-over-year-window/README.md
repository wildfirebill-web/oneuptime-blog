# How to Compute Year-Over-Year Growth with Window Functions in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Window Function, SQL, Analytics, LAG

Description: Use LAG() with a 12-month or 365-day offset to compute year-over-year revenue growth percentages directly in SQL without self-joins.

---

Year-over-year (YoY) growth is one of the most common business metrics: how does this period compare to the same period last year? In ClickHouse you can compute it cleanly using the `lag()` window function with an offset that steps back exactly one year worth of rows. No self-joins, no subqueries, just a single SELECT with a well-chosen partition and ordering.

## The Year-Over-Year Formula

```text
YoY Growth % = ((current_value - prior_year_value) / prior_year_value) * 100
```

The key is making `prior_year_value` available on the same row as `current_value`. `lag(metric, N)` returns the value from the row N positions before the current row inside the window. If rows are ordered by month and partitioned by a consistent dimension, an offset of 12 steps back exactly one year.

## Setting Up Monthly Revenue Data

```sql
CREATE TABLE monthly_revenue
(
    revenue_month Date,
    region        String,
    revenue       Float64
)
ENGINE = MergeTree()
ORDER BY (region, revenue_month);

INSERT INTO monthly_revenue VALUES
    ('2023-01-01', 'North', 42000),
    ('2023-02-01', 'North', 38000),
    ('2023-03-01', 'North', 51000),
    ('2023-04-01', 'North', 47000),
    ('2023-05-01', 'North', 55000),
    ('2023-06-01', 'North', 60000),
    ('2023-07-01', 'North', 58000),
    ('2023-08-01', 'North', 62000),
    ('2023-09-01', 'North', 57000),
    ('2023-10-01', 'North', 63000),
    ('2023-11-01', 'North', 70000),
    ('2023-12-01', 'North', 75000),
    ('2024-01-01', 'North', 48000),
    ('2024-02-01', 'North', 44000),
    ('2024-03-01', 'North', 59000),
    ('2024-04-01', 'North', 53000),
    ('2024-05-01', 'North', 61000),
    ('2024-06-01', 'North', 67000),
    ('2024-07-01', 'North', 65000),
    ('2024-08-01', 'North', 72000),
    ('2024-09-01', 'North', 66000),
    ('2024-10-01', 'North', 74000),
    ('2024-11-01', 'North', 82000),
    ('2024-12-01', 'North', 88000);
```

## Computing YoY Growth with LAG Offset 12

Partition by region so each region's months are ordered independently. Offset 12 steps back 12 months to the same month last year.

```sql
SELECT
    revenue_month,
    region,
    revenue,
    lag(revenue, 12) OVER (
        PARTITION BY region
        ORDER BY revenue_month
    ) AS prior_year_revenue,
    round(
        (revenue - lag(revenue, 12) OVER (
            PARTITION BY region
            ORDER BY revenue_month
        )) / lag(revenue, 12) OVER (
            PARTITION BY region
            ORDER BY revenue_month
        ) * 100,
        2
    ) AS yoy_growth_pct
FROM monthly_revenue
WHERE region = 'North'
ORDER BY revenue_month;
```

For the first 12 rows (all of 2023), `prior_year_revenue` is NULL and `yoy_growth_pct` is NULL because there is no prior year data. Starting from January 2024, both columns populate.

## Cleaner Version Using a Subquery

Repeating the `lag()` expression three times is verbose. Use a subquery to compute it once.

```sql
SELECT
    revenue_month,
    region,
    revenue,
    prior_year_revenue,
    round(
        (revenue - prior_year_revenue) / prior_year_revenue * 100,
        2
    ) AS yoy_growth_pct
FROM (
    SELECT
        revenue_month,
        region,
        revenue,
        lag(revenue, 12) OVER (
            PARTITION BY region
            ORDER BY revenue_month
        ) AS prior_year_revenue
    FROM monthly_revenue
)
WHERE prior_year_revenue IS NOT NULL
ORDER BY region, revenue_month;
```

The WHERE clause filters out the first year's rows where no comparison is possible, keeping the output clean.

## Multi-Region YoY Comparison

When multiple regions are present, PARTITION BY region ensures the lag stays within the same region.

```sql
INSERT INTO monthly_revenue VALUES
    ('2023-01-01', 'South', 31000), ('2024-01-01', 'South', 36000),
    ('2023-02-01', 'South', 29000), ('2024-02-01', 'South', 33000),
    ('2023-03-01', 'South', 35000), ('2024-03-01', 'South', 41000),
    ('2023-04-01', 'South', 33000), ('2024-04-01', 'South', 39000),
    ('2023-05-01', 'South', 37000), ('2024-05-01', 'South', 44000),
    ('2023-06-01', 'South', 40000), ('2024-06-01', 'South', 48000),
    ('2023-07-01', 'South', 38000), ('2024-07-01', 'South', 46000),
    ('2023-08-01', 'South', 42000), ('2024-08-01', 'South', 51000),
    ('2023-09-01', 'South', 39000), ('2024-09-01', 'South', 47000),
    ('2023-10-01', 'South', 44000), ('2024-10-01', 'South', 53000),
    ('2023-11-01', 'South', 50000), ('2024-11-01', 'South', 59000),
    ('2023-12-01', 'South', 55000), ('2024-12-01', 'South', 64000);
```

```sql
SELECT
    revenue_month,
    region,
    revenue,
    prior_year_revenue,
    round((revenue - prior_year_revenue) / prior_year_revenue * 100, 2) AS yoy_growth_pct
FROM (
    SELECT
        revenue_month,
        region,
        revenue,
        lag(revenue, 12) OVER (
            PARTITION BY region
            ORDER BY revenue_month
        ) AS prior_year_revenue
    FROM monthly_revenue
)
WHERE prior_year_revenue IS NOT NULL
ORDER BY region, revenue_month;
```

North and South are computed independently. January 2024 for North compares to January 2023 for North, and the same holds for South.

## Daily Data with LAG Offset 365

If your data is daily, use an offset of 365. Be aware that leap years mean the same calendar date is sometimes 366 days apart. For strict same-calendar-date comparison, consider joining on `date - interval 1 year` instead, but for smooth trend lines, 365 rows is often acceptable.

```sql
CREATE TABLE daily_revenue
(
    revenue_date  Date,
    revenue       Float64
)
ENGINE = MergeTree()
ORDER BY revenue_date;
```

```sql
SELECT
    revenue_date,
    revenue,
    lag(revenue, 365) OVER (ORDER BY revenue_date) AS prior_year_revenue,
    round(
        (revenue - lag(revenue, 365) OVER (ORDER BY revenue_date))
        / lag(revenue, 365) OVER (ORDER BY revenue_date) * 100,
        2
    ) AS yoy_growth_pct
FROM daily_revenue
ORDER BY revenue_date;
```

## Annotating Growth Direction

Add a label column to make reporting easier to read.

```sql
SELECT
    revenue_month,
    region,
    revenue,
    prior_year_revenue,
    yoy_growth_pct,
    CASE
        WHEN yoy_growth_pct > 0  THEN 'Growth'
        WHEN yoy_growth_pct < 0  THEN 'Decline'
        ELSE                          'Flat'
    END AS trend
FROM (
    SELECT
        revenue_month,
        region,
        revenue,
        lag(revenue, 12) OVER (
            PARTITION BY region
            ORDER BY revenue_month
        ) AS prior_year_revenue,
        round(
            (revenue - lag(revenue, 12) OVER (
                PARTITION BY region
                ORDER BY revenue_month
            )) / lag(revenue, 12) OVER (
                PARTITION BY region
                ORDER BY revenue_month
            ) * 100,
            2
        ) AS yoy_growth_pct
    FROM monthly_revenue
)
WHERE prior_year_revenue IS NOT NULL
ORDER BY region, revenue_month;
```

## Summary

Year-over-year growth in ClickHouse is best expressed with `lag()` using an offset equal to the number of rows that span one year (12 for monthly data, 365 for daily data). Partition by any dimension that should be compared independently, such as region or product category. Wrapping the lag in a subquery avoids repeating the expression when computing the percentage change. Filtering rows where `prior_year_revenue IS NOT NULL` produces a clean final result that starts only when a full year of history is available.
