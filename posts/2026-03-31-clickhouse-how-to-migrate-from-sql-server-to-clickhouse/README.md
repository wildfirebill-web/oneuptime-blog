# How to Migrate from SQL Server to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL Server, Migration, Analytics, Data Warehouse, ETL

Description: Migrate analytical workloads from Microsoft SQL Server to ClickHouse to reduce licensing costs and improve query performance on large datasets.

---

Microsoft SQL Server is a full-featured RDBMS, but its row-oriented storage is not optimized for large-scale analytical queries. Migrating analytical workloads to ClickHouse reduces licensing costs and dramatically improves aggregation performance.

## Step 1 - Export Data from SQL Server

Use `bcp` (Bulk Copy Program) to export to CSV:

```bash
bcp "SELECT * FROM analytics.dbo.page_views" \
  queryout /tmp/page_views.csv \
  -c -t"," -r"\n" \
  -S sqlserver-host \
  -U sa -P password
```

Or use SQL Server's EXPORT to a flat file wizard, or via SSMS.

For large tables, export in date-partitioned chunks:

```sql
DECLARE @month DATE = '2024-01-01';
WHILE @month < '2025-01-01'
BEGIN
  EXEC xp_cmdshell 'bcp "SELECT * FROM analytics.dbo.page_views WHERE created_at >= '''
    + CONVERT(VARCHAR, @month, 120) + ''' AND created_at < DATEADD(month,1,'''
    + CONVERT(VARCHAR, @month, 120) + ''')" queryout /tmp/pv_'
    + REPLACE(CONVERT(VARCHAR, @month, 120), '-', '') + '.csv -c -t, -S localhost -U sa -P pass';
  SET @month = DATEADD(month, 1, @month);
END
```

## Step 2 - Create the ClickHouse Table

Map SQL Server types to ClickHouse:

```sql
-- SQL Server DDL
-- CREATE TABLE page_views (
--     event_id     UNIQUEIDENTIFIER NOT NULL DEFAULT NEWID(),
--     user_id      NVARCHAR(100),
--     page         NVARCHAR(500),
--     duration_ms  INT DEFAULT 0,
--     created_at   DATETIME2 NOT NULL
-- );

-- ClickHouse equivalent
CREATE TABLE analytics.page_views
(
    event_id    UUID DEFAULT generateUUIDv4(),
    user_id     String,
    page        LowCardinality(String),
    duration_ms UInt32 DEFAULT 0,
    created_at  DateTime64(3)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(created_at)
ORDER BY (created_at, user_id);
```

## Step 3 - Load Data into ClickHouse

```bash
clickhouse-client \
  --query "INSERT INTO analytics.page_views FORMAT CSV" \
  < /tmp/page_views.csv
```

## Step 4 - Translate T-SQL to ClickHouse SQL

```sql
-- T-SQL: date functions
SELECT DATEPART(YEAR, created_at) AS year, COUNT(*) FROM page_views GROUP BY DATEPART(YEAR, created_at)

-- ClickHouse
SELECT toYear(created_at) AS year, count() FROM analytics.page_views GROUP BY year
```

```sql
-- T-SQL: TOP N
SELECT TOP 10 page, COUNT(*) AS views FROM page_views GROUP BY page ORDER BY views DESC

-- ClickHouse
SELECT page, count() AS views FROM analytics.page_views GROUP BY page ORDER BY views DESC LIMIT 10
```

```sql
-- T-SQL: ISNULL
SELECT ISNULL(user_id, 'anonymous') AS uid FROM page_views

-- ClickHouse
SELECT ifNull(user_id, 'anonymous') AS uid FROM analytics.page_views
```

## Step 5 - Handle Identity Columns

SQL Server `IDENTITY` columns have no direct equivalent. Use UInt64 with a sequence or UUID:

```sql
-- ClickHouse auto-increment alternative
CREATE TABLE analytics.orders (
    id UInt64,
    ...
) ENGINE = ReplacingMergeTree()
ORDER BY id;
```

## Summary

Migrating from SQL Server to ClickHouse for analytics involves exporting via `bcp`, creating a ClickHouse schema with equivalent types, and translating T-SQL functions to ClickHouse equivalents. The result is significantly faster aggregation queries and the elimination of SQL Server licensing costs.
