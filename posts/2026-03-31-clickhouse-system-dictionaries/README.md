# How to Use system.dictionaries to Monitor Dictionaries in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, system.dictionaries, Dictionary, Monitoring, Cache

Description: Learn how to use the system.dictionaries table in ClickHouse to monitor dictionary status, memory usage, load time, and last update timestamps.

---

ClickHouse dictionaries are in-memory key-value structures used to speed up lookups in queries. The `system.dictionaries` table provides operational visibility into all loaded dictionaries - their status, memory consumption, load time, and configuration details.

## What Is system.dictionaries?

The `system.dictionaries` table is a system table that exposes metadata and runtime statistics for every dictionary defined in ClickHouse. It helps you understand which dictionaries are loaded, how much memory they consume, when they were last updated, and whether any are in an error state.

## Basic Query

```sql
SELECT
    database,
    name,
    status,
    origin,
    type,
    source,
    bytes_allocated,
    element_count,
    last_successful_update_time,
    load_factor
FROM system.dictionaries
ORDER BY bytes_allocated DESC;
```

## Check Dictionary Status

Dictionaries can be in one of several states: `LOADED`, `FAILED`, `LOADING`, `LOADED_AND_RELOADING`. To find any that failed to load:

```sql
SELECT
    name,
    status,
    last_exception
FROM system.dictionaries
WHERE status != 'LOADED';
```

## Monitor Memory Usage

To see how much memory each dictionary occupies:

```sql
SELECT
    name,
    type,
    element_count,
    formatReadableSize(bytes_allocated) AS memory_used,
    load_factor
FROM system.dictionaries
WHERE status = 'LOADED'
ORDER BY bytes_allocated DESC;
```

High memory usage may indicate that a dictionary has too many keys or is using an inefficient layout type.

## Check When Dictionaries Were Last Updated

For dictionaries backed by external sources (MySQL, ClickHouse tables, files), it is important to know how fresh the data is:

```sql
SELECT
    name,
    source,
    last_successful_update_time,
    dateDiff('minute', last_successful_update_time, now()) AS minutes_since_update
FROM system.dictionaries
ORDER BY last_successful_update_time ASC;
```

## Monitor Load Duration

To identify dictionaries that are slow to load, which can delay server startup or background reloads:

```sql
SELECT
    name,
    source,
    loading_duration,
    element_count
FROM system.dictionaries
ORDER BY loading_duration DESC;
```

## Manually Reload a Dictionary

If you update the source data and want to refresh a dictionary immediately:

```sql
SYSTEM RELOAD DICTIONARY geo_regions;
```

After reloading, verify the status:

```sql
SELECT name, status, last_successful_update_time
FROM system.dictionaries
WHERE name = 'geo_regions';
```

## Useful Columns Reference

| Column | Description |
|---|---|
| `name` | Dictionary name |
| `status` | Current state (LOADED, FAILED, etc.) |
| `source` | Source type (MySQL, file, ClickHouse) |
| `bytes_allocated` | Memory used in bytes |
| `element_count` | Number of key-value entries |
| `load_factor` | Cache hit rate (for cache-type dicts) |
| `last_successful_update_time` | Timestamp of last successful reload |
| `last_exception` | Error message if status is FAILED |

## Summary

The `system.dictionaries` table provides essential operational metrics for all ClickHouse dictionaries. Use it to monitor memory footprint, detect failed loads, verify data freshness, and diagnose slow-loading dictionaries. Combine it with `SYSTEM RELOAD DICTIONARY` to manage dictionary lifecycle as part of your operational runbook.
