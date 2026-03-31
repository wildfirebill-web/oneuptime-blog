# How to Use SYSTEM RELOAD DICTIONARY in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, SQL, SYSTEM, Dictionary, Reload

Description: Learn how to use SYSTEM RELOAD DICTIONARY and SYSTEM RELOAD DICTIONARIES in ClickHouse to refresh dictionary data without restarting the server.

---

ClickHouse dictionaries are in-memory key-value structures loaded from external sources such as databases, files, or HTTP endpoints. They accelerate lookups in queries via functions like `dictGet`. When the underlying source data changes, ClickHouse does not automatically pick up updates unless the dictionary's `LIFETIME` interval triggers a reload - or you force one manually with `SYSTEM RELOAD DICTIONARY`. This post explains both the single-dictionary and all-dictionaries reload commands, how to check reload status, and common use cases.

## SYSTEM RELOAD DICTIONARY

Reloads a single named dictionary immediately, regardless of its `LIFETIME` setting.

```sql
SYSTEM RELOAD DICTIONARY my_database.country_codes;
```

If the dictionary is in the current database, you can omit the database prefix:

```sql
SYSTEM RELOAD DICTIONARY country_codes;
```

The command blocks until the reload completes or fails. If the external source is unavailable, ClickHouse keeps the old dictionary data and raises an error.

## SYSTEM RELOAD DICTIONARIES

Reloads all dictionaries that are currently loaded (have been accessed at least once or are set to `LOADABLE_ON_STARTUP`).

```sql
SYSTEM RELOAD DICTIONARIES;
```

This is useful after a bulk update to the underlying source data, or after deploying changes to multiple dictionary definitions.

## Creating a Dictionary to Test With

```sql
-- Source table
CREATE TABLE country_source
(
    code String,
    name String
)
ENGINE = MergeTree()
ORDER BY code;

INSERT INTO country_source VALUES
    ('US', 'United States'),
    ('GB', 'United Kingdom'),
    ('DE', 'Germany');

-- Dictionary backed by the source table
CREATE DICTIONARY country_codes
(
    code String,
    name String
)
PRIMARY KEY code
SOURCE(CLICKHOUSE(TABLE 'country_source'))
LAYOUT(HASHED())
LIFETIME(MIN 0 MAX 300); -- auto-reload every 0-300 seconds
```

## Using the Dictionary

```sql
SELECT dictGet('country_codes', 'name', 'US') AS country_name;
-- Returns: United States
```

Now update the source and force a reload:

```sql
INSERT INTO country_source VALUES ('FR', 'France');

SYSTEM RELOAD DICTIONARY country_codes;

SELECT dictGet('country_codes', 'name', 'FR') AS country_name;
-- Returns: France (available immediately after reload)
```

## Checking Dictionary Status

Use `system.dictionaries` to inspect loaded dictionaries, their status, and last reload time:

```sql
SELECT
    database,
    name,
    status,
    origin,
    element_count,
    bytes_allocated,
    last_successful_update_time,
    last_exception
FROM system.dictionaries
ORDER BY last_successful_update_time DESC;
```

Key status values:

| Status | Meaning |
|---|---|
| `LOADED` | Dictionary is loaded and available |
| `LOADING` | Reload in progress |
| `FAILED` | Last reload failed; old data still served |
| `NOT_LOADED` | Never loaded (accessed for the first time) |

## Handling Failed Reloads

If a reload fails (e.g., source database is down), ClickHouse logs the error and continues serving the last successfully loaded data:

```sql
-- Check for dictionaries with errors
SELECT
    name,
    status,
    last_exception
FROM system.dictionaries
WHERE last_exception != ''
ORDER BY name;
```

To retry after fixing the source:

```sql
SYSTEM RELOAD DICTIONARY country_codes;
```

## LIFETIME and Automatic Reloads

Dictionaries reload automatically based on their `LIFETIME` configuration. `MIN 0 MAX 300` means ClickHouse will reload the dictionary every 0 to 300 seconds (the exact interval is randomized within the range to spread load).

```sql
-- Dictionary with a 1-minute auto-reload window
CREATE DICTIONARY ip_allowlist
(
    ip_address String,
    allowed UInt8
)
PRIMARY KEY ip_address
SOURCE(CLICKHOUSE(TABLE 'ip_allowlist_source'))
LAYOUT(HASHED())
LIFETIME(MIN 60 MAX 60);
```

For dictionaries that should never auto-reload (only manual reloads):

```sql
LIFETIME(MIN 0 MAX 0) -- disables automatic reloading
```

## Common Use Cases

- **After updating reference data**: Reload a currency or country code dictionary after a source table update.
- **After deploying a new dictionary definition**: Force an immediate load without waiting for the first query to trigger it.
- **During maintenance**: Reload all dictionaries after restoring a source database from backup.
- **Debugging stale lookups**: If `dictGet` returns unexpected values, check `last_successful_update_time` and force a reload to rule out stale data.

## Summary

`SYSTEM RELOAD DICTIONARY` lets you manually refresh a single dictionary from its source, while `SYSTEM RELOAD DICTIONARIES` refreshes all loaded dictionaries at once. Combined with the `LIFETIME` setting for automatic reloads and the `system.dictionaries` table for status monitoring, these commands give you full control over dictionary freshness without requiring a server restart.
