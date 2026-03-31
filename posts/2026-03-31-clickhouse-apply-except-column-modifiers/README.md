# How to Use APPLY and EXCEPT Column Modifiers in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, APPLY Modifier, EXCEPT Modifier, Column Modifier, SQL

Description: Learn how the APPLY and EXCEPT column modifiers in ClickHouse let you transform or exclude columns from wildcard selections cleanly.

---

ClickHouse column modifiers `APPLY` and `EXCEPT` extend wildcard and `COLUMNS` expressions, letting you transform matched columns or exclude specific ones without enumerating them all.

## The EXCEPT Modifier

`EXCEPT` removes named columns from a wildcard match:

```sql
SELECT * EXCEPT (password_hash, internal_token) FROM users;
```

This returns every column except the two sensitive fields. It is cleaner than listing all other columns manually.

### EXCEPT with Multiple Columns

```sql
SELECT * EXCEPT (created_at, updated_at, deleted_at) FROM orders;
```

## The APPLY Modifier

`APPLY` applies a function to every column in a match:

```sql
SELECT COLUMNS('metric_.*') APPLY(round) FROM measurements;
```

All matched columns are passed through `round()` individually.

### APPLY with Multi-Argument Functions

For functions that take additional parameters, use a lambda:

```sql
SELECT COLUMNS('latency_.*') APPLY(x -> round(x, 2)) FROM requests;
```

## Combining APPLY and EXCEPT

You can chain modifiers:

```sql
SELECT * EXCEPT (id, ts) APPLY(toString) FROM raw_events;
```

This converts every non-metadata column to string, useful for CSV export or debugging.

## Practical Use Case: Normalizing Metrics

Suppose all your metric columns need to be divided by 1000 to convert from microseconds to milliseconds:

```sql
SELECT
    request_id,
    COLUMNS('.*_us') APPLY(x -> x / 1000.0) AS latency_ms
FROM traces;
```

## Practical Use Case: Masking PII

Exclude PII columns before exporting data to an analytics table:

```sql
INSERT INTO analytics.events
SELECT * EXCEPT (email, phone, ip_address) FROM raw.events;
```

## REPLACE Modifier Preview

A related modifier, `REPLACE`, substitutes the expression for a matched column while keeping its name - covered in a separate post.

## Summary

`APPLY` and `EXCEPT` are concise tools for post-selection transformation and exclusion in ClickHouse. Together with `COLUMNS`, they form a powerful system for writing flexible, maintainable analytical queries.
