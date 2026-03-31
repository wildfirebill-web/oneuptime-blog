# How to Use PARTITION BY in MySQL Window Functions

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Window Function, SQL, Analytics

Description: Learn how to use PARTITION BY in MySQL window functions to divide result sets into groups and apply calculations independently within each group.

---

## What Is PARTITION BY?

`PARTITION BY` is an optional clause in MySQL window functions that divides the result set into partitions (groups). The window function is then applied independently to each partition, resetting for each group - similar to `GROUP BY` but without collapsing the rows.

## Syntax

```sql
function() OVER (
    PARTITION BY column1 [, column2, ...]
    [ORDER BY column [ASC|DESC]]
    [ROWS|RANGE BETWEEN ...]
)
```

## Difference Between PARTITION BY and GROUP BY

```text
GROUP BY:     collapses rows into one row per group
PARTITION BY: keeps all rows but applies the function per group
```

```sql
-- GROUP BY: one row per department
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department;

-- PARTITION BY: all rows, with department average in each row
SELECT
    name,
    department,
    salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees;
```

## Sample Data

```sql
CREATE TABLE employees (
    name        VARCHAR(50),
    department  VARCHAR(50),
    hire_date   DATE,
    salary      DECIMAL(10,2)
);

INSERT INTO employees VALUES
  ('Alice',   'Engineering', '2020-01-15', 90000),
  ('Bob',     'Engineering', '2021-06-01', 80000),
  ('Carol',   'Engineering', '2019-03-20', 95000),
  ('Dave',    'Marketing',   '2020-07-10', 70000),
  ('Eve',     'Marketing',   '2022-01-05', 65000),
  ('Frank',   'HR',          '2021-09-15', 60000),
  ('Grace',   'HR',          '2023-02-28', 55000);
```

## Rank Within Each Department

```sql
SELECT
    name,
    department,
    salary,
    RANK() OVER (
        PARTITION BY department
        ORDER BY salary DESC
    ) AS dept_salary_rank
FROM employees
ORDER BY department, dept_salary_rank;
```

## Running Total Per Department

```sql
SELECT
    name,
    department,
    hire_date,
    salary,
    SUM(salary) OVER (
        PARTITION BY department
        ORDER BY hire_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS dept_running_total
FROM employees
ORDER BY department, hire_date;
```

## Department Average and Deviation

```sql
SELECT
    name,
    department,
    salary,
    ROUND(AVG(salary) OVER (PARTITION BY department), 2) AS dept_avg,
    ROUND(salary - AVG(salary) OVER (PARTITION BY department), 2) AS deviation
FROM employees
ORDER BY department, salary DESC;
```

## Row Number Per Partition

Assign a sequential number to employees in each department ordered by hire date:

```sql
SELECT
    name,
    department,
    hire_date,
    ROW_NUMBER() OVER (
        PARTITION BY department
        ORDER BY hire_date
    ) AS hire_order
FROM employees;
```

## First and Last Value Per Partition

```sql
SELECT
    name,
    department,
    salary,
    FIRST_VALUE(name) OVER (
        PARTITION BY department
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS top_earner,
    LAST_VALUE(name) OVER (
        PARTITION BY department
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS lowest_earner
FROM employees
ORDER BY department, salary DESC;
```

## PARTITION BY Multiple Columns

```sql
SELECT
    name,
    department,
    hire_date,
    YEAR(hire_date) AS hire_year,
    COUNT(*) OVER (
        PARTITION BY department, YEAR(hire_date)
    ) AS hires_same_year_dept
FROM employees
ORDER BY department, hire_year;
```

## Window Without PARTITION BY

Omitting `PARTITION BY` treats the entire result set as one partition:

```sql
SELECT
    name,
    salary,
    RANK() OVER (ORDER BY salary DESC) AS overall_rank,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees;
```

## Summary

`PARTITION BY` in MySQL window functions groups rows into independent partitions, allowing aggregate and ranking functions to operate per group without collapsing rows. It is the key clause that distinguishes window functions from `GROUP BY` aggregates. You can partition by single or multiple columns, and combine it with `ORDER BY` and frame specifications for advanced analytics like running totals, ranks, and moving averages per group.
