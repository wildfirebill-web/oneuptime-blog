# How to Use SELECT Statement in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL, DML, SELECT, Query

Description: Write MySQL SELECT queries to retrieve data, use column aliases, expressions, DISTINCT, and understand the full SELECT clause structure.

---

## How It Works

`SELECT` retrieves rows from one or more tables, views, or subqueries. MySQL processes the clauses in a specific logical order, even though they are written in a different order in the SQL text.

```mermaid
flowchart LR
    A[FROM / JOIN] --> B[WHERE]
    B --> C[GROUP BY]
    C --> D[HAVING]
    D --> E[SELECT expressions]
    E --> F[DISTINCT]
    F --> G[ORDER BY]
    G --> H[LIMIT / OFFSET]
```

## Syntax

```sql
SELECT [ALL | DISTINCT]
    expression [AS alias], ...
FROM table_name
[JOIN ...]
[WHERE condition]
[GROUP BY col, ...]
[HAVING condition]
[ORDER BY col [ASC|DESC], ...]
[LIMIT n [OFFSET m]];
```

## Setting Up the Sample Data

```sql
CREATE TABLE employees (
    id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50)    NOT NULL,
    last_name  VARCHAR(50)    NOT NULL,
    department VARCHAR(50)    NOT NULL,
    salary     DECIMAL(10, 2) NOT NULL,
    hired_on   DATE           NOT NULL
);

INSERT INTO employees (first_name, last_name, department, salary, hired_on) VALUES
    ('Alice',   'Smith',   'Engineering', 95000.00, '2021-03-15'),
    ('Bob',     'Jones',   'Marketing',   72000.00, '2020-07-01'),
    ('Carol',   'Williams','Engineering', 105000.00,'2019-11-20'),
    ('Dave',    'Brown',   'Sales',       60000.00, '2022-01-10'),
    ('Eve',     'Davis',   'Engineering', 88000.00, '2023-06-05'),
    ('Frank',   'Miller',  'Marketing',   78000.00, '2021-09-30'),
    ('Grace',   'Wilson',  'Sales',       65000.00, '2020-04-22');
```

## Selecting All Columns

```sql
SELECT * FROM employees;
```

```text
+----+------------+-----------+--------------+-----------+------------+
| id | first_name | last_name | department   | salary    | hired_on   |
+----+------------+-----------+--------------+-----------+------------+
|  1 | Alice      | Smith     | Engineering  |  95000.00 | 2021-03-15 |
...
```

Avoid `SELECT *` in production code - it returns unnecessary columns and breaks when columns are added or removed.

## Selecting Specific Columns

```sql
SELECT first_name, last_name, department, salary
FROM employees;
```

## Column Aliases

Use `AS` to give columns meaningful names in the result.

```sql
SELECT
    first_name AS "First Name",
    last_name  AS "Last Name",
    salary     AS "Annual Salary"
FROM employees;
```

## Computed Expressions

```sql
SELECT
    first_name,
    last_name,
    salary,
    salary / 12                           AS monthly_salary,
    CONCAT(first_name, ' ', last_name)    AS full_name,
    DATEDIFF(CURDATE(), hired_on) / 365.0 AS years_tenure
FROM employees;
```

```text
+------------+-----------+-----------+----------------+--------------+--------------+
| first_name | last_name | salary    | monthly_salary | full_name    | years_tenure |
+------------+-----------+-----------+----------------+--------------+--------------+
| Alice      | Smith     |  95000.00 |    7916.666667 | Alice Smith  |    3.044...  |
| Bob        | Jones     |  72000.00 |    6000.000000 | Bob Jones    |    4.909...  |
...
```

## DISTINCT - Remove Duplicate Rows

```sql
SELECT DISTINCT department FROM employees ORDER BY department;
```

```text
+-------------+
| department  |
+-------------+
| Engineering |
| Marketing   |
| Sales       |
+-------------+
```

## Selecting Constants and Functions

```sql
SELECT
    NOW()           AS current_time,
    CURDATE()       AS today,
    USER()          AS current_user,
    DATABASE()      AS current_db,
    VERSION()       AS mysql_version;
```

## Selecting Without a Table - Calculations

```sql
SELECT 2 + 2 AS result;
SELECT MD5('password') AS hash;
SELECT PI() AS pi_value;
```

## NULL Handling

```sql
-- IFNULL: return default if value is NULL
SELECT first_name, IFNULL(department, 'Unassigned') AS department
FROM employees;

-- COALESCE: return first non-NULL value
SELECT COALESCE(NULL, NULL, 'fallback') AS value;

-- IS NULL check
SELECT * FROM employees WHERE department IS NULL;
```

## Filtering with WHERE

```sql
SELECT first_name, last_name, salary
FROM employees
WHERE department = 'Engineering'
  AND salary > 90000;
```

```text
+------------+-----------+-----------+
| first_name | last_name | salary    |
+------------+-----------+-----------+
| Alice      | Smith     |  95000.00 |
| Carol      | Williams  | 105000.00 |
+------------+-----------+-----------+
```

## Sorting Results

```sql
SELECT first_name, last_name, salary
FROM employees
ORDER BY salary DESC
LIMIT 3;
```

```text
+------------+-----------+-----------+
| first_name | last_name | salary    |
+------------+-----------+-----------+
| Carol      | Williams  | 105000.00 |
| Alice      | Smith     |  95000.00 |
| Eve        | Davis     |  88000.00 |
+------------+-----------+-----------+
```

## Aggregation

```sql
SELECT
    department,
    COUNT(*)      AS headcount,
    AVG(salary)   AS avg_salary,
    MAX(salary)   AS max_salary,
    MIN(salary)   AS min_salary
FROM employees
GROUP BY department
ORDER BY avg_salary DESC;
```

```text
+-------------+-----------+-------------+------------+------------+
| department  | headcount | avg_salary  | max_salary | min_salary |
+-------------+-----------+-------------+------------+------------+
| Engineering |         3 | 96000.00000 | 105000.00  |  88000.00  |
| Marketing   |         2 | 75000.00000 |  78000.00  |  72000.00  |
| Sales       |         2 | 62500.00000 |  65000.00  |  60000.00  |
+-------------+-----------+-------------+------------+------------+
```

## Subquery in SELECT

```sql
SELECT
    first_name,
    salary,
    (SELECT AVG(salary) FROM employees) AS company_avg,
    salary - (SELECT AVG(salary) FROM employees) AS diff_from_avg
FROM employees
ORDER BY diff_from_avg DESC;
```

## Best Practices

- Always list specific columns rather than using `SELECT *` in production queries.
- Use meaningful column aliases (`AS`) for computed columns.
- Include a `LIMIT` clause when exploring data interactively to avoid accidentally fetching millions of rows.
- Use `EXPLAIN SELECT ...` to analyse the query execution plan before deploying slow queries.
- Avoid using functions on indexed columns in the WHERE clause (e.g., `WHERE YEAR(hired_on) = 2021`) as this prevents index usage. Rewrite as a range: `WHERE hired_on BETWEEN '2021-01-01' AND '2021-12-31'`.

## Summary

The `SELECT` statement is the foundation of all MySQL data retrieval. Its clauses are processed in a logical order: FROM, WHERE, GROUP BY, HAVING, SELECT, DISTINCT, ORDER BY, LIMIT. Use column aliases for readability, `DISTINCT` to remove duplicates, and aggregate functions with `GROUP BY` for summarisation. Always prefer explicit column lists over `SELECT *` in production code.
