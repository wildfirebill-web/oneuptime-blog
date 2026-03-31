# How to Migrate from SQL Server to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL Server, Migration, Analytics, OLAP, Data Warehouse

Description: Migrate analytical workloads from Microsoft SQL Server to ClickHouse using BCP export, ODBC, or bulk CSV transfer with schema translation guidance.

---

SQL Server is a capable OLTP database, but running heavy analytical queries on it taxes your operational workload. ClickHouse handles the same analytical queries orders of magnitude faster. This guide covers moving data and translating common T-SQL patterns.

## Export Options Overview

1. BCP (Bulk Copy Program) - fast native export
2. ODBC table engine - live query without moving data
3. SSMS / bcp CSV export - simple one-time migration

## Option 1 - ODBC Table Engine (Live Access)

Configure an ODBC DSN for SQL Server, then create a ClickHouse table:

```sql
CREATE TABLE sqlserver_source
ENGINE = ODBC('DSN=SQLServerDSN', 'dbo', 'events');
```

Then copy to ClickHouse:

```sql
INSERT INTO events SELECT * FROM sqlserver_source;
```

## Option 2 - BCP Export

Export to CSV using `bcp`:

```bash
bcp "SELECT * FROM mydb.dbo.events" queryout events.csv \
  -c -t, -r "\n" \
  -S SQLSERVER_HOST -U sa -P password
```

Add column headers by prepending them:

```bash
bcp mydb.dbo.events out events_noheader.csv -c -t, -S SQLSERVER_HOST -U sa -P password

# Get headers separately
sqlcmd -S SQLSERVER_HOST -U sa -P password -Q \
  "SET NOCOUNT ON; SELECT COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME='events' ORDER BY ORDINAL_POSITION" \
  -h -1 | tr '\n' ',' > header.csv

cat header.csv events_noheader.csv > events.csv
```

## Translating the Schema

SQL Server DDL:

```sql
CREATE TABLE events (
    event_id    BIGINT IDENTITY(1,1) PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    event_type  NVARCHAR(64),
    occurred_at DATETIME2,
    amount      DECIMAL(18, 4)
);
```

ClickHouse equivalent:

```sql
CREATE TABLE events (
    event_id    UInt64,
    user_id     UInt64,
    event_type  LowCardinality(String),
    occurred_at DateTime,
    amount      Decimal(18, 4)
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(occurred_at)
ORDER BY (occurred_at, user_id);
```

## Loading CSV into ClickHouse

```bash
clickhouse-client \
  --query "INSERT INTO events FORMAT CSVWithNames" \
  < events.csv
```

## Query Translation

T-SQL `DATEPART` and `GETDATE()`:

```sql
SELECT
    DATEPART(HOUR, occurred_at) AS hour,
    COUNT(*) AS cnt
FROM events
WHERE occurred_at >= DATEADD(DAY, -7, GETDATE())
GROUP BY DATEPART(HOUR, occurred_at)
ORDER BY hour;
```

ClickHouse equivalent:

```sql
SELECT
    toHour(occurred_at) AS hour,
    count() AS cnt
FROM events
WHERE occurred_at >= now() - INTERVAL 7 DAY
GROUP BY hour
ORDER BY hour;
```

T-SQL `TOP N` becomes `LIMIT N`:

```sql
-- T-SQL
SELECT TOP 10 user_id, SUM(amount) AS total FROM events GROUP BY user_id ORDER BY total DESC;

-- ClickHouse
SELECT user_id, sum(amount) AS total FROM events GROUP BY user_id ORDER BY total DESC LIMIT 10;
```

## Common T-SQL to ClickHouse Function Map

| T-SQL | ClickHouse |
|---|---|
| `GETDATE()` | `now()` |
| `DATEADD(DAY, -7, ts)` | `ts - INTERVAL 7 DAY` |
| `ISNULL(col, 0)` | `ifNull(col, 0)` |
| `CAST(col AS FLOAT)` | `toFloat64(col)` |
| `LEN(str)` | `length(str)` |

## Summary

Migrating from SQL Server to ClickHouse is well-supported by the BCP export tool and the ODBC table engine. Most T-SQL analytical queries translate directly with minor function name changes, making ClickHouse a drop-in analytical backend for SQL Server shops.
