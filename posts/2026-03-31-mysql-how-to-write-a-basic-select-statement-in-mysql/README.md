# How to Write a Basic SELECT Statement in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SELECT, SQL Basics, Querying, Database

Description: Learn how to write basic SELECT statements in MySQL to retrieve data from tables, using column selection, filtering, sorting, and limiting results.

---

## The SELECT Statement

The `SELECT` statement is the foundation of SQL querying. It retrieves data from one or more tables. The most basic form selects all columns from a table:

```sql
-- Select all columns from a table
SELECT * FROM employees;

-- Select specific columns
SELECT first_name, last_name, salary FROM employees;
```

## Setting Up Sample Data

```sql
CREATE TABLE employees (
    id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    department VARCHAR(50),
    salary DECIMAL(10, 2),
    hire_date DATE,
    is_active TINYINT(1) DEFAULT 1
);

INSERT INTO employees (first_name, last_name, department, salary, hire_date) VALUES
('Alice', 'Johnson', 'Engineering', 95000, '2020-03-15'),
('Bob', 'Smith', 'Marketing', 72000, '2019-07-01'),
('Carol', 'White', 'Engineering', 105000, '2018-11-20'),
('Dave', 'Brown', 'HR', 68000, '2021-01-10'),
('Eve', 'Davis', 'Engineering', 88000, '2022-05-18');
```

## Selecting Specific Columns

```sql
-- Select two columns
SELECT first_name, department FROM employees;

-- Use column aliases with AS
SELECT
    first_name AS "First Name",
    last_name AS "Last Name",
    salary AS "Annual Salary"
FROM employees;

-- Expressions and calculations in SELECT
SELECT
    first_name,
    salary,
    salary / 12 AS monthly_salary,
    salary * 0.08 AS estimated_tax
FROM employees;
```

## Using Literals and Functions in SELECT

```sql
-- String concatenation
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM employees;

-- Date functions
SELECT
    first_name,
    hire_date,
    YEAR(hire_date) AS hire_year,
    DATEDIFF(CURDATE(), hire_date) AS days_employed
FROM employees;

-- String functions
SELECT
    UPPER(first_name) AS first_upper,
    LOWER(last_name) AS last_lower,
    LENGTH(first_name) AS name_length
FROM employees;
```

## Filtering with WHERE

The `WHERE` clause filters rows before they are returned:

```sql
-- Basic comparison
SELECT * FROM employees WHERE department = 'Engineering';

-- Numeric comparison
SELECT first_name, salary FROM employees WHERE salary > 80000;

-- Multiple conditions with AND
SELECT * FROM employees
WHERE department = 'Engineering' AND salary > 90000;

-- Multiple conditions with OR
SELECT * FROM employees
WHERE department = 'HR' OR salary < 70000;
```

## Sorting Results with ORDER BY

```sql
-- Sort ascending (default)
SELECT first_name, salary FROM employees ORDER BY salary;

-- Sort descending
SELECT first_name, salary FROM employees ORDER BY salary DESC;

-- Sort by multiple columns
SELECT first_name, department, salary
FROM employees
ORDER BY department ASC, salary DESC;

-- Sort by an alias
SELECT first_name, salary * 12 AS annual_total
FROM employees
ORDER BY annual_total DESC;
```

## Limiting Results with LIMIT

```sql
-- Get the first 3 rows
SELECT * FROM employees LIMIT 3;

-- Get the top earners
SELECT first_name, salary FROM employees
ORDER BY salary DESC
LIMIT 3;

-- Pagination: skip first 2 rows, get next 3 (LIMIT offset, count)
SELECT * FROM employees
ORDER BY id
LIMIT 2, 3;

-- Alternative LIMIT with OFFSET syntax
SELECT * FROM employees
ORDER BY id
LIMIT 3 OFFSET 2;
```

## Removing Duplicates with DISTINCT

```sql
-- Get unique department names
SELECT DISTINCT department FROM employees;

-- Distinct across multiple columns
SELECT DISTINCT department, is_active FROM employees;
```

## Full SELECT Clause Order

The clauses of a SELECT statement must appear in this order:

```sql
SELECT   [DISTINCT] column_list
FROM     table_name
WHERE    condition
GROUP BY column_list
HAVING   condition
ORDER BY column_list [ASC | DESC]
LIMIT    [offset,] row_count;

-- Example using all common clauses together
SELECT
    department,
    COUNT(*) AS headcount,
    AVG(salary) AS avg_salary
FROM employees
WHERE is_active = 1
GROUP BY department
HAVING avg_salary > 70000
ORDER BY avg_salary DESC
LIMIT 5;
```

## Summary

The `SELECT` statement is the most fundamental SQL command for retrieving data from MySQL tables. Start with `SELECT columns FROM table`, then add `WHERE` to filter, `ORDER BY` to sort, and `LIMIT` to control result size. Use column aliases, expressions, and built-in functions to shape the output. Understanding the order of clauses and how each one affects the result set is essential for writing effective queries.
