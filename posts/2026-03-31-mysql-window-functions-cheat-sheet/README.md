# MySQL Window Functions Cheat Sheet

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Window Function, Query, Cheat Sheet

Description: Quick reference for MySQL window functions including ROW_NUMBER, RANK, DENSE_RANK, LAG, LEAD, NTILE, and frame clauses with practical SQL examples.

---

## Window Function Syntax

```sql
function_name() OVER (
  [PARTITION BY col1, col2]
  [ORDER BY col3 ASC|DESC]
  [frame_clause]
)
```

## Ranking Functions

```sql
SELECT name, salary, dept_id,
  ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS row_num,
  RANK()       OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rank_pos,
  DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dense_rank_pos
FROM employees;
```

```text
ROW_NUMBER  - unique sequential number, no ties
RANK        - ties share rank, gaps after ties (1,1,3)
DENSE_RANK  - ties share rank, no gaps         (1,1,2)
```

## NTILE (bucketing)

```sql
-- Divide employees into 4 salary quartiles
SELECT name, salary,
  NTILE(4) OVER (ORDER BY salary) AS quartile
FROM employees;
```

## PERCENT_RANK and CUME_DIST

```sql
SELECT name, salary,
  PERCENT_RANK() OVER (ORDER BY salary) AS pct_rank,  -- 0 to 1
  CUME_DIST()    OVER (ORDER BY salary) AS cume_dist  -- cumulative %
FROM employees;
```

## LAG and LEAD

Access values from preceding or following rows.

```sql
SELECT order_date, revenue,
  LAG(revenue, 1, 0)  OVER (ORDER BY order_date) AS prev_revenue,
  LEAD(revenue, 1, 0) OVER (ORDER BY order_date) AS next_revenue,
  revenue - LAG(revenue) OVER (ORDER BY order_date)  AS day_delta
FROM daily_sales;
```

## FIRST_VALUE and LAST_VALUE

```sql
SELECT name, salary, dept_id,
  FIRST_VALUE(salary) OVER (
    PARTITION BY dept_id ORDER BY salary DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS dept_max_salary
FROM employees;
```

## NTH_VALUE

```sql
-- Second highest salary per department
SELECT name, dept_id,
  NTH_VALUE(salary, 2) OVER (
    PARTITION BY dept_id ORDER BY salary DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS second_highest
FROM employees;
```

## Aggregate Window Functions

```sql
-- Running total
SELECT order_date, amount,
  SUM(amount) OVER (ORDER BY order_date) AS running_total,
  AVG(amount) OVER (ORDER BY order_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS rolling_7day_avg
FROM daily_revenue;
```

## Frame Clauses

```text
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
ROWS BETWEEN CURRENT ROW AND 2 FOLLOWING
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
RANGE BETWEEN INTERVAL 7 DAY PRECEDING AND CURRENT ROW
```

## Named Windows

```sql
SELECT name, salary,
  RANK()       OVER w AS rnk,
  DENSE_RANK() OVER w AS drnk,
  ROW_NUMBER() OVER w AS rnum
FROM employees
WINDOW w AS (PARTITION BY dept_id ORDER BY salary DESC);
```

## Top-N Per Group

```sql
-- Top 3 highest-paid employees per department
SELECT *
FROM (
  SELECT name, dept_id, salary,
    ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
  FROM employees
) ranked
WHERE rn <= 3;
```

## Summary

MySQL window functions (available since 8.0) perform calculations across a set of rows related to the current row without collapsing the result set. Use ROW_NUMBER/RANK/DENSE_RANK for ordering and deduplication, LAG/LEAD for time-series comparisons, and SUM/AVG OVER ORDER BY for running totals and rolling averages. Named windows reduce repetition in complex queries.
