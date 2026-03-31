# How to Calculate Running Totals in ClickHouse Without Window Functions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Running Total, Cumulative Sum, Aggregation, Array

Description: Learn how to calculate running totals in ClickHouse without window functions using arrayCumSum, self-join, and subquery-based cumulative aggregations.

---

## Running Totals in ClickHouse

While ClickHouse supports window functions (`SUM() OVER (...)`), older versions or specific use cases benefit from alternative approaches. Using `arrayCumSum`, self-joins, or runningAccumulate provides running totals without window function overhead.

## Sample Daily Revenue Table

```sql
CREATE TABLE daily_revenue
(
    day Date,
    revenue Float64
)
ENGINE = MergeTree()
ORDER BY day;

INSERT INTO daily_revenue VALUES
('2024-01-01', 1000),
('2024-01-02', 1200),
('2024-01-03', 900),
('2024-01-04', 1500),
('2024-01-05', 800);
```

## Method 1 - Window Function (Modern ClickHouse)

The simplest approach on ClickHouse 21.1+:

```sql
SELECT
    day,
    revenue,
    sum(revenue) OVER (ORDER BY day ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM daily_revenue
ORDER BY day;
```

## Method 2 - arrayCumSum

Aggregate into arrays first, then apply cumulative sum:

```sql
SELECT
    arrayJoin(days) AS day,
    arrayJoin(revenues) AS revenue,
    arrayJoin(running_totals) AS running_total
FROM (
    SELECT
        groupArray(day) AS days,
        groupArray(revenue) AS revenues,
        arrayCumSum(groupArray(revenue)) AS running_totals
    FROM (SELECT day, revenue FROM daily_revenue ORDER BY day)
);
```

## Method 3 - runningAccumulate

`runningAccumulate` applies an aggregate function cumulatively over sorted rows:

```sql
SELECT
    day,
    revenue,
    runningAccumulate(sum_state) AS running_total
FROM (
    SELECT
        day,
        revenue,
        sumState(revenue) AS sum_state
    FROM daily_revenue
    GROUP BY day, revenue
    ORDER BY day
);
```

## Method 4 - Self-Join Cumulative Sum

Classic triangular self-join approach:

```sql
SELECT
    d1.day,
    d1.revenue,
    sum(d2.revenue) AS running_total
FROM daily_revenue d1
JOIN daily_revenue d2 ON d2.day <= d1.day
GROUP BY d1.day, d1.revenue
ORDER BY d1.day;
```

## Running Count and Running Average

```sql
SELECT
    arrayJoin(days) AS day,
    arrayJoin(revenues) AS daily_revenue,
    arrayJoin(arrayCumSum(revenues)) AS cumulative_revenue,
    arrayJoin(arrayCumSum(ones)) AS row_number,
    arrayJoin(arrayCumSum(revenues)) / arrayJoin(arrayCumSum(ones)) AS running_avg
FROM (
    SELECT
        groupArray(day) AS days,
        groupArray(revenue) AS revenues,
        groupArray(1) AS ones
    FROM (SELECT day, revenue FROM daily_revenue ORDER BY day)
);
```

## Running Totals by Group

```sql
SELECT
    category,
    day,
    revenue,
    sum(revenue) OVER (PARTITION BY category ORDER BY day) AS category_running_total
FROM category_revenue
ORDER BY category, day;
```

## Summary

ClickHouse calculates running totals via window `SUM() OVER`, `arrayCumSum` on grouped arrays, or `runningAccumulate` with aggregate states. `arrayCumSum` is the most portable approach for older ClickHouse versions, while window functions are cleaner and preferred in ClickHouse 21.1+.
