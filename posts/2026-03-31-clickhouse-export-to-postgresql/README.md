# How to Export ClickHouse Data to PostgreSQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, PostgreSQL, Export, Data Transfer, postgresql Table Function

Description: Learn how to export ClickHouse query results to PostgreSQL using the postgresql table function and custom ETL scripts for cross-database data transfers.

---

Exporting ClickHouse analytical results back to PostgreSQL is useful for populating reporting tables, syncing aggregated metrics, and feeding OLTP systems with pre-computed analytics.

## Using the postgresql Table Function

ClickHouse can write directly to PostgreSQL tables:

```sql
INSERT INTO FUNCTION postgresql(
    'postgres:5432',
    'mydb',
    'daily_stats',
    'postgres_user',
    'postgres_password'
)
SELECT
    toDate(ts) AS date,
    event_type,
    count() AS event_count,
    uniq(user_id) AS unique_users
FROM events
WHERE toDate(ts) = today() - 1
GROUP BY date, event_type;
```

## Creating the Target Table in PostgreSQL

```sql
-- In PostgreSQL
CREATE TABLE daily_stats (
    date DATE,
    event_type VARCHAR(50),
    event_count BIGINT,
    unique_users INT
);

CREATE UNIQUE INDEX ON daily_stats(date, event_type);
```

## Upsert Pattern

PostgreSQL's `ON CONFLICT DO UPDATE` handles re-runs without duplicates. Use a staging table:

```sql
-- Export to a staging table first
INSERT INTO FUNCTION postgresql(
    'postgres:5432', 'mydb', 'daily_stats_staging',
    'postgres_user', 'postgres_password'
)
SELECT ...;

-- Then upsert in PostgreSQL
-- (run via psql or application code)
INSERT INTO daily_stats SELECT * FROM daily_stats_staging
ON CONFLICT (date, event_type)
DO UPDATE SET
    event_count = EXCLUDED.event_count,
    unique_users = EXCLUDED.unique_users;
```

## Exporting via Python

For complex transformations:

```python
import clickhouse_driver
import psycopg2

ch = clickhouse_driver.Client('clickhouse')
pg = psycopg2.connect("host=postgres dbname=mydb user=postgres password=secret")

rows = ch.execute("""
    SELECT toDate(ts), event_type, count(), uniq(user_id)
    FROM events
    WHERE toDate(ts) = today() - 1
    GROUP BY 1, 2
""")

with pg.cursor() as cur:
    cur.executemany("""
        INSERT INTO daily_stats VALUES (%s, %s, %s, %s)
        ON CONFLICT (date, event_type) DO UPDATE
        SET event_count = EXCLUDED.event_count
    """, rows)
pg.commit()
```

## Using the PostgreSQL Dictionary

For bidirectional access, define ClickHouse dictionaries sourced from PostgreSQL:

```sql
CREATE DICTIONARY product_lookup (
    product_id UInt32,
    product_name String
) PRIMARY KEY product_id
SOURCE(POSTGRESQL(
    host 'postgres' port 5432 user 'postgres' password 'secret'
    db 'mydb' table 'products'
))
LAYOUT(HASHED())
LIFETIME(300);
```

## Summary

Export ClickHouse results to PostgreSQL using the `postgresql` table function for direct INSERT, or Python with psycopg2 for upserts and complex transformations. Use staging tables to safely re-run exports without duplicating data.
