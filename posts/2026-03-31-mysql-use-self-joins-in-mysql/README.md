# How to Use Self Joins in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Self Join, Hierarchical Data, Sql, Database

Description: Learn how to use self joins in MySQL to query a table against itself for hierarchical data, employee-manager relationships, and sequential comparisons.

---

## Introduction

A self join in MySQL joins a table to itself. This is useful when a table has a column that references another row in the same table, such as an employee-manager relationship, a category hierarchy, or a linked list structure. To distinguish the two references, you must use table aliases.

## Basic Self Join Syntax

```sql
SELECT a.column1, b.column2
FROM table_name a
JOIN table_name b ON a.some_column = b.other_column;
```

Both `a` and `b` refer to the same table but represent different "instances" of it.

## Employee-Manager Hierarchy

A classic use case: employees table where each employee has a `manager_id` referencing another employee's `id`.

```sql
CREATE TABLE employees (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  manager_id INT,
  salary DECIMAL(10,2)
);
```

Find each employee with their manager's name:

```sql
SELECT
  e.name AS employee,
  m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id
ORDER BY m.name, e.name;
```

Use `LEFT JOIN` to include employees at the top of the hierarchy (who have no manager).

## Find Top-Level Managers

Employees who are not managed by anyone:

```sql
SELECT e.id, e.name
FROM employees e
WHERE NOT EXISTS (
  SELECT 1 FROM employees m WHERE m.id = e.manager_id
);
```

Or using self join:

```sql
SELECT e.id, e.name
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id
WHERE m.id IS NULL;
```

## Find Direct Reports

List all employees under a specific manager:

```sql
SELECT e.name AS employee
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE m.name = 'Sarah Chen';
```

## Category Hierarchy

A categories table where subcategories reference a parent category:

```sql
SELECT
  child.name AS subcategory,
  parent.name AS parent_category
FROM categories child
LEFT JOIN categories parent ON child.parent_id = parent.id
ORDER BY parent.name, child.name;
```

## Comparing Adjacent Rows

Self joins are useful for comparing a row to neighboring rows.

Find employees earning more than their manager:

```sql
SELECT
  e.name AS employee,
  e.salary AS employee_salary,
  m.name AS manager,
  m.salary AS manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

## Finding Pairs

Self joins can find all pairs of rows meeting a condition (e.g., coworkers in the same department):

```sql
SELECT
  a.name AS employee1,
  b.name AS employee2,
  a.department
FROM employees a
JOIN employees b ON a.department = b.department
  AND a.id < b.id  -- prevents duplicates and self-pairing
ORDER BY a.department, a.name;
```

The `a.id < b.id` condition ensures each pair appears only once and a person is not paired with themselves.

## Recursive Hierarchies

For deep hierarchies (more than one level), MySQL 8.0+ supports recursive CTEs:

```sql
WITH RECURSIVE org_chart AS (
  -- Anchor: top-level employees
  SELECT id, name, manager_id, 0 AS level
  FROM employees
  WHERE manager_id IS NULL

  UNION ALL

  -- Recursive: each employee and their subordinates
  SELECT e.id, e.name, e.manager_id, oc.level + 1
  FROM employees e
  JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT level, name FROM org_chart ORDER BY level, name;
```

## Performance Tips

- Index the foreign key column (e.g., `manager_id`) for fast self join lookups.
- Use `LEFT JOIN` when you want to include rows without a matching reference.

```sql
CREATE INDEX idx_employees_manager_id ON employees(manager_id);
```

## Summary

Self joins allow a MySQL table to be joined against itself, enabling queries on hierarchical or relational data within a single table. Common use cases include employee-manager hierarchies, category trees, and row comparison queries. Always use table aliases to distinguish the two references and index the join column for performance.
