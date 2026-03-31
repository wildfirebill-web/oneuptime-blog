# How to Fix "Cannot insert NULL into non-nullable column" in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, NULL, Schema, Troubleshooting, Data Quality

Description: Fix ClickHouse "Cannot insert NULL into non-nullable column" errors by handling NULL values at the application layer or adjusting column definitions.

---

## Understanding the Error

ClickHouse enforces strict nullability at the schema level. When you try to insert NULL into a column defined without `Nullable(...)`, you get:

```text
DB::Exception: Cannot insert NULL value into non-nullable column 'user_id'. (BAD_ARGUMENTS)
```

Unlike some databases that silently insert a default value, ClickHouse rejects the operation outright unless you configure it otherwise.

## Diagnosing the Issue

### Check Column Nullability

```sql
-- Find non-nullable columns in your table
SELECT name, type, default_kind, default_expression
FROM system.columns
WHERE table = 'events' AND database = 'analytics'
ORDER BY position;

-- Columns with type like 'UInt64' (not 'Nullable(UInt64)') are non-nullable
```

### Identify the Failing Insert

```bash
# Check recent insert errors
clickhouse-client --query "
SELECT
    query_id,
    event_time,
    exception,
    left(query, 200) AS q
FROM system.query_log
WHERE exception LIKE '%NULL%non-nullable%'
  AND event_time > now() - INTERVAL 1 HOUR
ORDER BY event_time DESC
LIMIT 10
"
```

## Fix 1 - Provide a Default Value Instead of NULL

The simplest fix is to replace NULLs with a sensible default before inserting:

```sql
-- Replace NULL user_id with 0 using COALESCE
INSERT INTO analytics.events
SELECT
    COALESCE(user_id, 0) AS user_id,
    event_type,
    event_time
FROM external_source;
```

```python
import clickhouse_driver

client = clickhouse_driver.Client('localhost')

rows = [
    (row['user_id'] or 0, row['event_type'], row['event_time'])
    for row in raw_data
]

client.execute('INSERT INTO analytics.events (user_id, event_type, event_time) VALUES', rows)
```

## Fix 2 - Use input_format_null_as_default

For bulk data loads where you want NULLs to become the column default:

```sql
SET input_format_null_as_default = 1;

INSERT INTO analytics.events FORMAT CSVWithNames
user_id,event_type,event_time
\N,click,2024-01-15 10:00:00
123,view,2024-01-15 10:01:00
```

## Fix 3 - Change the Column to Nullable

If NULL is a valid business value for this column:

```sql
-- Change a non-nullable column to nullable
ALTER TABLE analytics.events
MODIFY COLUMN user_id Nullable(UInt64);

-- Verify the change
DESCRIBE TABLE analytics.events;
```

Note: Adding Nullable increases storage and query overhead. Avoid it for high-cardinality columns in the ORDER BY key.

## Fix 4 - Add a Column Default

Set a column-level DEFAULT so NULLs from CSV/JSON are substituted automatically:

```sql
-- Add default value so NULL inserts use the default
ALTER TABLE analytics.events
MODIFY COLUMN user_id UInt64 DEFAULT 0;
```

## Fix 5 - Handle at the Source System

For application code, validate before inserting:

```javascript
// Node.js example - sanitize before insert
const sanitized = events.map(e => ({
  user_id: e.user_id ?? 0,
  event_type: e.event_type ?? 'unknown',
  event_time: e.event_time ?? new Date().toISOString()
}));

await client.insert({
  table: 'analytics.events',
  values: sanitized,
  format: 'JSONEachRow'
});
```

## Handling NULLs in Downstream Queries

When you do use Nullable columns, handle NULLs in queries explicitly:

```sql
-- Check for NULL
SELECT count()
FROM analytics.events
WHERE user_id IS NULL;

-- Replace NULL in output
SELECT
    ifNull(user_id, 0) AS user_id,
    event_type
FROM analytics.events;

-- Filter out NULLs
SELECT user_id, count()
FROM analytics.events
WHERE user_id IS NOT NULL
GROUP BY user_id;
```

## Summary

ClickHouse strictly enforces non-nullable column constraints and rejects inserts that include NULL values for such columns. The cleanest fix is replacing NULLs with defaults using COALESCE or `input_format_null_as_default` in bulk loads. If NULL is a meaningful value in your domain, change the column to `Nullable(T)` but be aware of the performance trade-off. Always validate data at the application or ETL layer to catch nullability violations before they reach ClickHouse.
