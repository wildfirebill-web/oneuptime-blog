# How to Design One-to-Many Relationships in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Relationship, Schema Design, Foreign Key

Description: Learn how to implement one-to-many relationships in MySQL with foreign keys, cascade rules, and efficient join queries.

---

A one-to-many relationship is the most common relational pattern: one row in a parent table corresponds to multiple rows in a child table. A blog with many posts, a customer with many orders, a department with many employees - these are all classic examples.

## Schema Design

Place the foreign key on the "many" side - the child table. The parent table needs no changes beyond its primary key.

```sql
CREATE TABLE departments (
    id   INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE employees (
    id            INT UNSIGNED NOT NULL AUTO_INCREMENT,
    department_id INT UNSIGNED NOT NULL,
    name          VARCHAR(150) NOT NULL,
    hired_on      DATE         NOT NULL,
    PRIMARY KEY (id),
    KEY idx_dept (department_id),
    CONSTRAINT fk_emp_dept
        FOREIGN KEY (department_id) REFERENCES departments (id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE
);
```

The `KEY idx_dept` index on `department_id` is critical - without it, every join or foreign key check performs a full table scan on `employees`.

## Choosing Cascade Behavior

| Action | Effect |
|---|---|
| `ON DELETE RESTRICT` | Block deletion of parent if children exist |
| `ON DELETE CASCADE` | Delete children automatically when parent is deleted |
| `ON DELETE SET NULL` | Set foreign key to NULL when parent is deleted |

Use `RESTRICT` for important business entities like orders. Use `CASCADE` for detail rows like order line items.

## Querying One-to-Many

```sql
-- All employees grouped by department
SELECT d.name AS department, COUNT(e.id) AS headcount
FROM   departments d
LEFT JOIN employees e ON e.department_id = d.id
GROUP BY d.id, d.name
ORDER BY headcount DESC;
```

```sql
-- Employees hired in the last 90 days per department
SELECT d.name, e.name, e.hired_on
FROM   departments d
JOIN   employees e ON e.department_id = d.id
WHERE  e.hired_on >= CURDATE() - INTERVAL 90 DAY;
```

## Handling Nullable Parent

When an employee may not belong to a department, make the foreign key nullable:

```sql
ALTER TABLE employees
    MODIFY department_id INT UNSIGNED NULL,
    DROP FOREIGN KEY fk_emp_dept,
    ADD CONSTRAINT fk_emp_dept
        FOREIGN KEY (department_id) REFERENCES departments (id)
        ON DELETE SET NULL
        ON UPDATE CASCADE;
```

## Counting Children Efficiently

```sql
SELECT d.name,
       COUNT(e.id) AS emp_count
FROM   departments d
LEFT JOIN employees e ON e.department_id = d.id
GROUP BY d.id;
```

Avoid running a correlated subquery like `(SELECT COUNT(*) FROM employees WHERE department_id = d.id)` per row - a single join with `GROUP BY` is far more efficient.

## Summary

One-to-many relationships belong to the child table as a non-unique foreign key. Always index the foreign key column for join performance. Choose `ON DELETE` behavior based on business rules - `RESTRICT` protects data integrity while `CASCADE` automates cleanup of dependent rows. Use `LEFT JOIN` with `COUNT` to list all parents even those with no children.
