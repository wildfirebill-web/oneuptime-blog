# How to Use Recursive CTEs in MySQL 8

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Recursive CTE, Hierarchical Data, SQL, Trees

Description: Learn how to write recursive CTEs in MySQL 8 to traverse hierarchical data like organizational charts, category trees, and bill of materials.

---

## What Are Recursive CTEs

A recursive CTE uses `WITH RECURSIVE` and consists of two parts joined by `UNION ALL`:
1. The anchor member - the starting rows of the recursion.
2. The recursive member - a query that references the CTE itself to continue the recursion.

MySQL 8 supports recursive CTEs natively, making it straightforward to process tree and graph structures without stored procedures.

## Basic Syntax

```sql
WITH RECURSIVE cte_name AS (
  -- Anchor: starting rows
  SELECT ...

  UNION ALL

  -- Recursive: join CTE back to itself
  SELECT ... FROM source_table JOIN cte_name ON ...
)
SELECT * FROM cte_name;
```

## Example Table - Employee Hierarchy

```sql
CREATE TABLE employees (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  manager_id INT,
  FOREIGN KEY (manager_id) REFERENCES employees(id)
);

INSERT INTO employees VALUES
  (1, 'Alice', NULL),    -- CEO
  (2, 'Bob', 1),         -- VP Engineering
  (3, 'Carol', 1),       -- VP Sales
  (4, 'Dave', 2),        -- Engineer
  (5, 'Eve', 2),         -- Engineer
  (6, 'Frank', 3),       -- Sales Rep
  (7, 'Grace', 3);       -- Sales Rep
```

## Traversing the Hierarchy from Top Down

```sql
WITH RECURSIVE org_chart AS (
  -- Anchor: start with the CEO (no manager)
  SELECT id, name, manager_id, 0 AS depth, CAST(name AS CHAR(1000)) AS path
  FROM employees
  WHERE manager_id IS NULL

  UNION ALL

  -- Recursive: find each employee's direct reports
  SELECT e.id, e.name, e.manager_id, oc.depth + 1, CONCAT(oc.path, ' > ', e.name)
  FROM employees e
  JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT id, CONCAT(REPEAT('  ', depth), name) AS indented_name, depth, path
FROM org_chart
ORDER BY path;
```

Sample output:

```text
+----+------------------+-------+------------------------------+
| id | indented_name    | depth | path                         |
+----+------------------+-------+------------------------------+
|  1 | Alice            |     0 | Alice                        |
|  2 |   Bob            |     1 | Alice > Bob                  |
|  4 |     Dave         |     2 | Alice > Bob > Dave           |
|  5 |     Eve          |     2 | Alice > Bob > Eve            |
|  3 |   Carol          |     1 | Alice > Carol                |
|  6 |     Frank        |     2 | Alice > Carol > Frank        |
|  7 |     Grace        |     2 | Alice > Carol > Grace        |
+----+------------------+-------+------------------------------+
```

## Finding All Ancestors (Bottom-Up)

```sql
WITH RECURSIVE ancestors AS (
  -- Anchor: start with a specific employee
  SELECT id, name, manager_id
  FROM employees
  WHERE id = 5  -- Starting from Eve

  UNION ALL

  -- Recursive: find the manager of each row
  SELECT e.id, e.name, e.manager_id
  FROM employees e
  JOIN ancestors a ON e.id = a.manager_id
)
SELECT * FROM ancestors;
```

## Category Tree Example

```sql
CREATE TABLE categories (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  parent_id INT
);

INSERT INTO categories VALUES
  (1, 'Electronics', NULL),
  (2, 'Computers', 1),
  (3, 'Smartphones', 1),
  (4, 'Laptops', 2),
  (5, 'Desktops', 2);
```

```sql
WITH RECURSIVE category_tree AS (
  SELECT id, name, parent_id, 0 AS level,
         CAST(id AS CHAR(200)) AS id_path
  FROM categories
  WHERE parent_id IS NULL

  UNION ALL

  SELECT c.id, c.name, c.parent_id, ct.level + 1,
         CONCAT(ct.id_path, ',', c.id)
  FROM categories c
  JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT CONCAT(REPEAT('-- ', level), name) AS category, level, id_path
FROM category_tree
ORDER BY id_path;
```

## Generating a Number Series

Recursive CTEs are also useful for generating sequences:

```sql
WITH RECURSIVE numbers AS (
  SELECT 1 AS n
  UNION ALL
  SELECT n + 1 FROM numbers WHERE n < 10
)
SELECT n FROM numbers;
```

## Controlling Recursion Depth

MySQL has a `cte_max_recursion_depth` system variable (default 1000). For deep trees you may need to increase it:

```sql
SET SESSION cte_max_recursion_depth = 5000;
```

To prevent infinite loops in circular data, always include a depth counter and WHERE condition:

```sql
WHERE oc.depth < 10
```

## Summary

Recursive CTEs in MySQL 8 elegantly solve hierarchical data traversal problems that previously required multiple round trips or stored procedures. By combining an anchor query with a recursive member using `UNION ALL`, you can traverse organization charts, category trees, bill-of-materials structures, and any parent-child relationship stored in a self-referencing table.
