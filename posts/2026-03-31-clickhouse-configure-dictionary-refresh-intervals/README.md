# How to Configure Dictionary Refresh Intervals in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Dictionary, Refresh Interval, LIFETIME, Configuration

Description: Learn how to configure ClickHouse dictionary refresh intervals using LIFETIME settings to balance data freshness with source system load.

---

## Why Refresh Intervals Matter

Dictionary data can become stale if the source table or file is updated. ClickHouse refreshes dictionaries according to the `LIFETIME` setting. Choosing the right interval balances data freshness with query stability and source system load.

## Basic LIFETIME Configuration

Use a fixed interval (in seconds):

```sql
CREATE DICTIONARY my_dict (...)
PRIMARY KEY id
SOURCE(...)
LAYOUT(HASHED())
LIFETIME(3600);  -- Refresh every 3600 seconds (1 hour)
```

## MIN/MAX LIFETIME for Randomized Refresh

Using MIN/MAX distributes refresh load across time to avoid all ClickHouse nodes refreshing simultaneously:

```sql
LIFETIME(MIN 300 MAX 600)
-- Each replica picks a random refresh time between 5-10 minutes
```

This is the recommended approach for clustered deployments.

## Disable Automatic Refresh

Use `LIFETIME(0)` to load once at startup and never refresh:

```sql
LIFETIME(0)
-- Dictionary loads once and never auto-refreshes
-- Must be reloaded manually with SYSTEM RELOAD DICTIONARY
```

This is appropriate for static reference data like country codes.

## Refresh Strategies by Data Type

```text
Data type              | Suggested LIFETIME
Static (country codes) | 0 (manual refresh only)
Slow-changing (plans)  | MIN 3600 MAX 7200 (1-2 hours)
Fast-changing (prices) | MIN 60 MAX 120 (1-2 minutes)
Real-time (segments)   | MIN 30 MAX 60 (30-60 seconds)
```

## Monitor Refresh Timing

```sql
SELECT
    name,
    last_successful_update_time,
    last_exception,
    loading_duration,
    status,
    bytes_allocated
FROM system.dictionaries
ORDER BY last_successful_update_time DESC;
```

## Force Manual Reload

```sql
SYSTEM RELOAD DICTIONARY my_dict;
```

Reload all dictionaries at once:

```sql
SYSTEM RELOAD DICTIONARIES;
```

## Dictionary with Invalidation Check

For ClickHouse sources, use `INVALIDATE_QUERY` to only reload when source data changes:

```sql
CREATE DICTIONARY product_dict (
    product_id UInt64,
    name String DEFAULT ''
)
PRIMARY KEY product_id
SOURCE(CLICKHOUSE(
    DB 'default'
    TABLE 'products'
    INVALIDATE_QUERY 'SELECT max(updated_at) FROM products'
))
LAYOUT(HASHED())
LIFETIME(MIN 60 MAX 120);
```

ClickHouse runs the invalidation query on each refresh cycle. If the result hasn't changed, the full reload is skipped.

## MySQL INVALIDATE_QUERY Example

```sql
SOURCE(MYSQL(
    HOST 'mysql.internal'
    PORT 3306
    USER 'reader'
    PASSWORD 'secret'
    DB 'catalog'
    TABLE 'products'
    INVALIDATE_QUERY 'SELECT UNIX_TIMESTAMP(MAX(updated_at)) FROM products'
))
```

## Check Reload Duration

Long reload times indicate the source query is slow or the dictionary is large:

```sql
SELECT
    name,
    loading_duration,
    element_count,
    formatReadableSize(bytes_allocated)
FROM system.dictionaries
WHERE loading_duration > 10;
```

## Summary

Dictionary refresh intervals in ClickHouse are controlled by the `LIFETIME` setting. Use MIN/MAX intervals for distributed deployments, `LIFETIME(0)` for static data, and `INVALIDATE_QUERY` to avoid unnecessary reloads when source data hasn't changed. Monitor `system.dictionaries` to track reload times and catch failures early.
