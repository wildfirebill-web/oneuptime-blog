# How to Build an Organizational Chart with Recursive CTEs in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Recursive CTE, SQL, Hierarchical, Query

Description: Learn how to query employee reporting structures and build organizational charts in MySQL using recursive CTEs to traverse manager-subordinate relationships.

---

## Storing an Org Chart in MySQL

The simplest way to model a reporting hierarchy is an adjacency list: each employee row stores the `manager_id` of their direct manager. The top-level CEO has a `NULL` manager.

```sql
CREATE TABLE employees (
  employee_id   INT PRIMARY KEY,
  manager_id    INT NULL,
  full_name     VARCHAR(100),
  title         VARCHAR(100),
  department    VARCHAR(50)
);

INSERT INTO employees VALUES
  (1,  NULL, 'Alice Chen',    'CEO',               'Executive'),
  (2,  1,    'Bob Kumar',     'VP Engineering',    'Engineering'),
  (3,  1,    'Carol Smith',   'VP Sales',          'Sales'),
  (4,  2,    'Dan Lee',       'Engineering Mgr',   'Engineering'),
  (5,  2,    'Eve Torres',    'Engineering Mgr',   'Engineering'),
  (6,  3,    'Frank Brown',   'Sales Mgr',         'Sales'),
  (7,  4,    'Grace Park',    'Senior Engineer',   'Engineering'),
  (8,  4,    'Hank Wilson',   'Engineer',          'Engineering'),
  (9,  5,    'Iris Patel',    'Senior Engineer',   'Engineering'),
  (10, 6,    'Jack Nguyen',   'Account Executive', 'Sales');
```

## Building the Full Org Chart

Use `WITH RECURSIVE` to walk from the CEO down through every level:

```sql
WITH RECURSIVE org_chart AS (
  -- Anchor: CEO (no manager)
  SELECT
    employee_id,
    manager_id,
    full_name,
    title,
    department,
    0 AS level,
    CAST(full_name AS CHAR(1000)) AS reporting_chain
  FROM employees
  WHERE manager_id IS NULL

  UNION ALL

  -- Recursive: direct reports of each level
  SELECT
    e.employee_id,
    e.manager_id,
    e.full_name,
    e.title,
    e.department,
    oc.level + 1,
    CONCAT(oc.reporting_chain, ' -> ', e.full_name)
  FROM employees e
  INNER JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT
  CONCAT(REPEAT('    ', level), full_name) AS indented_name,
  title,
  department,
  level,
  reporting_chain
FROM org_chart
ORDER BY reporting_chain;
```

## Counting Direct and Indirect Reports

A recursive CTE makes it easy to count everyone who reports to a given manager, at any depth:

```sql
WITH RECURSIVE subordinates AS (
  SELECT employee_id, manager_id, full_name
  FROM employees
  WHERE manager_id = 2  -- Bob Kumar (VP Engineering)

  UNION ALL

  SELECT e.employee_id, e.manager_id, e.full_name
  FROM employees e
  INNER JOIN subordinates s ON e.manager_id = s.employee_id
)
SELECT COUNT(*) AS total_subordinates FROM subordinates;
```

## Finding the Chain of Command for an Employee

To show everyone in the reporting chain above a specific employee:

```sql
WITH RECURSIVE chain AS (
  SELECT employee_id, manager_id, full_name, title, 0 AS level
  FROM employees
  WHERE employee_id = 8  -- Hank Wilson

  UNION ALL

  SELECT e.employee_id, e.manager_id, e.full_name, e.title, c.level + 1
  FROM employees e
  INNER JOIN chain c ON e.employee_id = c.manager_id
)
SELECT level, full_name, title
FROM chain
ORDER BY level;
```

## Finding Peers (Same Manager)

To find all direct reports of the same manager as an employee:

```sql
SELECT e.employee_id, e.full_name, e.title
FROM employees e
WHERE e.manager_id = (
  SELECT manager_id FROM employees WHERE employee_id = 8
)
AND e.employee_id <> 8;
```

## Span of Control Report

The span of control shows how many direct reports each manager has:

```sql
SELECT
  m.employee_id AS manager_id,
  m.full_name   AS manager_name,
  m.title,
  COUNT(e.employee_id) AS direct_reports
FROM employees m
LEFT JOIN employees e ON e.manager_id = m.employee_id
GROUP BY m.employee_id, m.full_name, m.title
HAVING direct_reports > 0
ORDER BY direct_reports DESC;
```

## Performance Tip

Add a foreign key and index on `manager_id` to speed up recursive joins:

```sql
ALTER TABLE employees
  ADD CONSTRAINT fk_manager FOREIGN KEY (manager_id)
    REFERENCES employees (employee_id),
  ADD INDEX idx_manager_id (manager_id);
```

## Summary

Recursive CTEs in MySQL 8 are the natural tool for org chart queries. The adjacency list model is simple to maintain, and `WITH RECURSIVE` handles top-down traversal, bottom-up chain-of-command lookups, and subtree reporting counts. Combined with the `level` and path columns built during recursion, you can produce indented org chart output directly in SQL.
