# How to Migrate from SQL Server to MySQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, SQL Server, Migration, SSMA, Database

Description: Migrate a Microsoft SQL Server database to MySQL by converting schema with SSMA, handling T-SQL differences, and transferring data reliably.

---

## Key Differences Between SQL Server and MySQL

| SQL Server Feature | MySQL Equivalent |
|---|---|
| `IDENTITY(1,1)` | `AUTO_INCREMENT` |
| `GETDATE()` | `NOW()` |
| `ISNULL(x, y)` | `IFNULL(x, y)` |
| `TOP N` | `LIMIT N` |
| `NVARCHAR` | `VARCHAR` with `utf8mb4` |
| `DATETIME2` | `DATETIME(6)` |
| `BIT` | `TINYINT(1)` or `BOOLEAN` |
| T-SQL stored procedures | MySQL stored procedures |
| `GO` batch separator | Not needed in MySQL |

## Step 1 - Schema Conversion with SSMA

Microsoft's SQL Server Migration Assistant (SSMA) for MySQL is a free tool that automates schema conversion:

```text
1. Download SSMA for MySQL from Microsoft
2. Connect to SQL Server as source
3. Connect to MySQL as target
4. Run Assessment Report
5. Convert Schema
6. Migrate Data
```

SSMA generates a migration report highlighting objects that need manual intervention.

## Step 2 - Manual Schema Fixes

Convert IDENTITY columns:

```sql
-- SQL Server
CREATE TABLE orders (
  id INT IDENTITY(1,1) PRIMARY KEY,
  customer_id INT NOT NULL,
  total MONEY NOT NULL
);

-- MySQL
CREATE TABLE orders (
  id INT AUTO_INCREMENT PRIMARY KEY,
  customer_id INT NOT NULL,
  total DECIMAL(19,4) NOT NULL
);
```

Convert NVARCHAR to VARCHAR with utf8mb4:

```sql
-- SQL Server
name NVARCHAR(100) NOT NULL

-- MySQL
name VARCHAR(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NOT NULL
```

## Step 3 - Rewriting T-SQL Functions

Common T-SQL to MySQL function conversions:

```sql
-- SQL Server: string functions
LEN(col)           -> CHAR_LENGTH(col)
CHARINDEX(x, y)    -> LOCATE(x, y)
SUBSTRING(s, 1, 5) -> SUBSTRING(s, 1, 5)  -- same
CONVERT(VARCHAR, d, 120) -> DATE_FORMAT(d, '%Y-%m-%d %H:%i:%s')

-- SQL Server: date functions
DATEADD(day, 7, NOW())  -> DATE_ADD(NOW(), INTERVAL 7 DAY)
DATEDIFF(day, d1, d2)   -> DATEDIFF(d2, d1)  -- note reversed order
YEAR(d), MONTH(d), DAY(d) -> YEAR(d), MONTH(d), DAY(d)  -- same
```

## Step 4 - Converting T-SQL Stored Procedures

SQL Server T-SQL:

```sql
CREATE PROCEDURE GetOrdersByCustomer
  @CustomerID INT
AS
BEGIN
  SET NOCOUNT ON;
  SELECT * FROM orders WHERE customer_id = @CustomerID;
END;
```

MySQL equivalent:

```sql
DELIMITER $$
CREATE PROCEDURE GetOrdersByCustomer(IN p_CustomerID INT)
BEGIN
  SELECT * FROM orders WHERE customer_id = p_CustomerID;
END$$
DELIMITER ;
```

## Step 5 - Data Export from SQL Server

```bash
# Export using bcp (Bulk Copy Program)
bcp mydb.dbo.orders out /tmp/orders.dat -c -t',' -r'\n' \
  -S sqlserver.example.com -U sa -P 'Password!'
```

Import into MySQL:

```sql
LOAD DATA INFILE '/tmp/orders.dat'
INTO TABLE orders
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
(id, customer_id, total, created_at);
```

## Step 6 - Validate Row Counts and Checksums

```sql
-- In MySQL
SELECT TABLE_NAME, TABLE_ROWS
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'myapp'
ORDER BY TABLE_NAME;
```

Compare against SQL Server:

```sql
-- In SQL Server
SELECT t.name, p.rows
FROM sys.tables t
JOIN sys.partitions p ON t.object_id = p.object_id
WHERE p.index_id < 2
ORDER BY t.name;
```

## Summary

Migrating from SQL Server to MySQL requires converting IDENTITY columns to AUTO_INCREMENT, replacing T-SQL functions with MySQL equivalents, rewriting stored procedures without T-SQL syntax, and handling NVARCHAR/unicode with utf8mb4. SSMA automates most schema conversion, but stored procedures and complex T-SQL logic require manual review and rewriting.
