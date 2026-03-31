# How to Handle Dictionary Loading Failures in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Dictionary, Error Handling, Reliability, Troubleshooting

Description: Learn how ClickHouse handles dictionary loading failures, how to diagnose errors, and how to build resilient queries that tolerate missing dictionary data.

---

Dictionary loading failures happen when the source database is unreachable, the file is missing, the query fails, or the data contains schema mismatches. Understanding how ClickHouse handles these failures helps you build resilient analytics pipelines.

## Default Failure Behavior

When a reload fails, ClickHouse retains the last successfully loaded version and continues serving queries from it. The failure is recorded in `system.dictionaries`.

## Diagnosing Failures

```sql
SELECT
    name,
    status,
    last_exception,
    loading_start_time,
    last_successful_update_time
FROM system.dictionaries
WHERE status != 'LOADED';
```

Common status values:
- `LOADED` - successfully loaded and ready
- `FAILED` - initial load failed; no data available
- `LOADING` - currently being loaded
- `FAILED_AND_RELOADING` - previous load failed; trying again

## What Happens on Initial Load Failure

If a dictionary has never been loaded successfully (e.g., the source was unreachable when ClickHouse started), queries using `dictGet` will throw an exception:

```text
Exception: Dictionary 'currency_dict' not loaded
```

Use `dictGetOrDefault` to provide a safe fallback:

```sql
SELECT
    order_id,
    dictGetOrDefault('currency_dict', 'rate_to_usd', currency_code, toFloat64(1.0)) AS rate
FROM orders;
```

## Using dictHas for Safe Lookups

```sql
SELECT
    order_id,
    currency_code,
    if(
        dictHas('currency_dict', currency_code),
        dictGet('currency_dict', 'rate_to_usd', currency_code),
        1.0
    ) AS rate
FROM orders;
```

## Forcing a Reload After Fixing the Source

```sql
SYSTEM RELOAD DICTIONARY currency_dict;
```

Check the result immediately:

```sql
SELECT name, status, last_exception
FROM system.dictionaries
WHERE name = 'currency_dict';
```

## Configuring on_error Behavior (XML Config)

In `config.xml` or a custom dictionary XML file, you can set `on_error` to control behavior on initial load failure:

```text
<on_error>throw</on_error>
```

or

```text
<on_error>default_value</on_error>
```

`default_value` returns the attribute's default value instead of throwing an exception.

## Monitoring with Metrics

ClickHouse exposes metrics for dictionary failures:

```sql
SELECT metric, value
FROM system.metrics
WHERE metric LIKE '%Dictionary%';
```

## Setting Up Alerts

Use ClickHouse's built-in alerting or external monitoring to watch for:
- `status = 'FAILED'` in `system.dictionaries`
- Long time since `last_successful_update_time`

```sql
SELECT name
FROM system.dictionaries
WHERE last_successful_update_time < now() - INTERVAL 30 MINUTE
  AND status != 'LOADING';
```

## Source-Side Best Practices

- Use a read-only database user with minimal permissions
- Test the source query manually before defining the dictionary
- Ensure network connectivity between ClickHouse and the source on startup

## Summary

ClickHouse keeps the last good dictionary version when a reload fails, so queries continue working. Monitor `system.dictionaries` for failures, use `dictGetOrDefault` to handle missing entries gracefully, and set up alerts on stale `last_successful_update_time` to catch persistent source connectivity issues early.
