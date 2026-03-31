# How to Use Self Joins in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Self Join, SQL, Hierarchical Data, Querying

Description: Learn how to use self joins in MySQL to query hierarchical and relational data within the same table using table aliases and practical examples.

---

## What Is a Self Join?

A self join is when a table is joined with itself. This is useful for querying hierarchical data (like an employee-manager relationship), comparing rows within the same table, or finding related records in a single table. You must use table aliases to distinguish the two instances of the table.

```sql
-- Basic self join structure
SELECT a.column, b.column
FROM table_name a
JOIN table_name b ON a.some_id = b.other_id;
```

## Employee-Manager Hierarchy

The classic self join use case is an organizational hierarchy where each employee has a `manager_id` that references another employee in the same table:

```sql
CREATE TABLE employees (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    job_title VARCHAR(100),
    manager_id INT,
    salary DECIMAL(10, 2),
    FOREIGN KEY (manager_id) REFERENCES employees(id)
);

INSERT INTO employees (id, name, job_title, manager_id, salary) VALUES
(1, 'Alice CEO', 'CEO', NULL, 200000),
(2, 'Bob VP', 'VP Engineering', 1, 150000),
(3, 'Carol VP', 'VP Marketing', 1, 140000),
(4, 'Dave Eng', 'Engineer', 2, 95000),
(5, 'Eve Eng', 'Engineer', 2, 98000),
(6, 'Frank Mkt', 'Marketer', 3, 75000);

-- Get each employee with their manager's name
SELECT
    e.name AS employee,
    e.job_title,
    m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

## Finding Employees Earning More Than Their Manager

```sql
-- Self join to compare employee salary with manager salary
SELECT e.name AS employee, e.salary AS emp_salary,
       m.name AS manager, m.salary AS mgr_salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

## Multi-Level Hierarchy

```sql
-- Get employees with both manager and skip-level manager (grandmanager)
SELECT
    e.name AS employee,
    m.name AS manager,
    gm.name AS skip_level_manager
FROM employees e
LEFT JOIN employees m  ON e.manager_id = m.id
LEFT JOIN employees gm ON m.manager_id = gm.id;
```

## Finding Pairs of Rows (Combinatorial)

Self joins can enumerate all pairs of rows:

```sql
-- Find all pairs of products in the same category
SELECT
    a.name AS product1,
    b.name AS product2,
    a.category
FROM products a
JOIN products b ON a.category = b.category AND a.id < b.id
ORDER BY a.category, product1;

-- Find customers in the same city
SELECT
    c1.name AS customer1,
    c2.name AS customer2,
    c1.city
FROM customers c1
JOIN customers c2 ON c1.city = c2.city AND c1.id < c2.id;
```

## Comparing Sequential Rows

```sql
-- Compare each sale with the previous day's sale (self join on date offset)
SELECT
    s1.sale_date,
    s1.revenue AS today_revenue,
    s2.revenue AS yesterday_revenue,
    s1.revenue - s2.revenue AS day_over_day_change
FROM daily_revenue s1
JOIN daily_revenue s2
  ON s2.sale_date = DATE_SUB(s1.sale_date, INTERVAL 1 DAY)
ORDER BY s1.sale_date;
```

## Finding Duplicate Records

```sql
-- Find duplicate emails in a customer table
SELECT a.id, a.email, b.id AS duplicate_id
FROM customers a
JOIN customers b ON a.email = b.email AND a.id < b.id;

-- Count how many duplicates exist per email
SELECT email, COUNT(*) AS copies
FROM customers
GROUP BY email
HAVING copies > 1;
```

## Self Join with Aggregation

```sql
-- Find employees whose salary is above the average for their department
SELECT e.name, e.department, e.salary
FROM employees e
JOIN (
    SELECT department, AVG(salary) AS dept_avg
    FROM employees
    GROUP BY department
) dept_stats ON e.department = dept_stats.department
WHERE e.salary > dept_stats.dept_avg;
```

## Summary

Self joins are essential for working with hierarchical data, finding related records within a single table, and comparing rows against each other. Always use meaningful table aliases to distinguish the two instances. Use `LEFT JOIN` for hierarchy queries to include root nodes (like the CEO with no manager). For deep hierarchies, consider using recursive CTEs instead of multiple self joins.
