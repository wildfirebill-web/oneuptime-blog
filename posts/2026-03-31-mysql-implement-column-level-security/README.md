# How to Implement Column-Level Security in MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Security, Column-Level Security, View, Privilege

Description: Restrict access to sensitive MySQL columns using column-level privileges, views, and masking techniques to protect PII and confidential data.

---

## Column-Level Security in MySQL

MySQL supports column-level privileges natively, allowing you to grant SELECT, INSERT, or UPDATE on specific columns rather than entire tables. This is useful for protecting PII fields like social security numbers, salaries, or payment card data.

## Using Column-Level GRANT

```sql
CREATE TABLE employees (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100),
  department VARCHAR(50),
  salary DECIMAL(10,2),
  ssn CHAR(11)
);
```

Grant a reporting user access to non-sensitive columns only:

```sql
GRANT SELECT (id, name, department) ON myapp.employees TO 'reporting_user'@'%';
```

The user can query those columns:

```sql
-- Works
SELECT id, name, department FROM employees;

-- Fails
SELECT salary FROM employees;
-- ERROR 1143: SELECT command denied to user for column 'salary'
```

## Checking Column Privileges

```sql
SELECT TABLE_NAME, COLUMN_NAME, PRIVILEGE_TYPE, GRANTEE
FROM information_schema.COLUMN_PRIVILEGES
WHERE TABLE_SCHEMA = 'myapp'
ORDER BY GRANTEE, TABLE_NAME, COLUMN_NAME;
```

## Using Views for Column Masking

Column-level GRANT prevents access but reveals that the column exists. Views let you mask or transform sensitive data:

```sql
-- Full data view for privileged users
CREATE VIEW employees_full AS
SELECT id, name, department, salary, ssn FROM employees;

-- Masked view for general users
CREATE VIEW employees_masked AS
SELECT
  id,
  name,
  department,
  NULL AS salary,
  CONCAT('***-**-', RIGHT(ssn, 4)) AS ssn_masked
FROM employees;
```

Grant accordingly:

```sql
GRANT SELECT ON myapp.employees_full TO 'hr_admin'@'%';
GRANT SELECT ON myapp.employees_masked TO 'reporting_user'@'%';
REVOKE ALL ON myapp.employees FROM 'reporting_user'@'%';
```

## Dynamic Column Masking with Stored Functions

For more flexible masking, use a function:

```sql
DELIMITER $$

CREATE FUNCTION mask_ssn(ssn CHAR(11))
RETURNS CHAR(11)
DETERMINISTIC
BEGIN
  RETURN CONCAT('***-**-', RIGHT(ssn, 4));
END$$

DELIMITER ;
```

Use it in a view:

```sql
CREATE VIEW employees_safe AS
SELECT id, name, department, mask_ssn(ssn) AS ssn FROM employees;
```

## Protecting Columns on INSERT and UPDATE

Restrict write access to specific columns:

```sql
-- Allow updating department but not salary
GRANT UPDATE (department) ON myapp.employees TO 'dept_manager'@'%';
```

Attempting to update salary:

```sql
UPDATE employees SET salary = 99999 WHERE id = 1;
-- ERROR 1143: UPDATE command denied to user for column 'salary'
```

## Auditing Column Access

Use the MySQL audit plugin or general log to track column-level access patterns:

```sql
SET GLOBAL general_log = 'ON';
SET GLOBAL general_log_file = '/var/log/mysql/general.log';
```

Then review the log:

```bash
grep "salary\|ssn" /var/log/mysql/general.log
```

Disable after auditing to avoid performance impact:

```sql
SET GLOBAL general_log = 'OFF';
```

## Summary

MySQL column-level security uses a combination of native column GRANT/REVOKE, views that expose only permitted columns, and masking functions that transform sensitive values before returning them. For any table containing PII or financial data, grant application users access through purpose-built views rather than directly on the base table, and use column-level grants as an additional backstop.
