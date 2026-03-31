# How to Use format() for String Formatting in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, String Function, Formatting, Analytics, Reporting

Description: Learn how format() uses Python-style {} placeholders in ClickHouse to build formatted messages, generate readable output, and compose strings from column values.

---

ClickHouse's `format()` function provides Python-style string interpolation using `{}` placeholders. Rather than concatenating strings and cast expressions manually with the `||` operator, `format()` lets you write a readable template and pass the substitution arguments in order. It is particularly useful for building log messages, generating human-readable report lines, and composing dynamic strings from multiple column values.

## Function Signature

```text
format(template, arg1, arg2, ...)
```

Each `{}` in `template` is replaced in order by the string representation of the corresponding argument. ClickHouse converts each argument to `String` automatically, so you do not need to call `toString()` explicitly for most types.

```sql
-- Basic positional substitution
SELECT format('Hello, {}! You have {} messages.', 'Alice', 42) AS greeting;
-- result: 'Hello, Alice! You have 42 messages.'
```

```sql
-- Values from table columns are substituted in order
SELECT format('{} {} {}', first_name, last_name, email) AS formatted_user
FROM users
LIMIT 5;
```

## Generating Formatted Messages

Building structured log or alert messages from table columns is a natural fit for `format()`:

```sql
-- Compose an alert message from event fields
SELECT
    event_id,
    format(
        '[{}] {} on host {} - latency {}ms exceeds threshold {}ms',
        toDateTime(event_time),
        event_type,
        host_name,
        latency_ms,
        threshold_ms
    ) AS alert_message
FROM latency_events
WHERE latency_ms > threshold_ms
ORDER BY event_time DESC
LIMIT 10;
```

```sql
-- Build a human-readable error summary line per row
SELECT
    format(
        'Error {} in {} at line {}: {}',
        error_code,
        file_name,
        line_number,
        error_message
    ) AS error_summary
FROM compile_errors
ORDER BY file_name, line_number
LIMIT 20;
```

## Creating Human-Readable Output from Column Values

Analytical query results often need to be presented in a readable format for downstream consumers. `format()` keeps the template readable in the SQL source:

```sql
-- Format a duration stored as seconds into a readable string
SELECT
    service,
    format(
        'p50={}ms  p90={}ms  p99={}ms',
        round(p50_ms),
        round(p90_ms),
        round(p99_ms)
    ) AS latency_summary
FROM service_stats
ORDER BY service
LIMIT 10;
```

```sql
-- Build a one-line summary for each user account
SELECT
    user_id,
    format(
        '{} {} ({}), joined {}, {} orders, ${} total spend',
        first_name,
        last_name,
        email,
        toDate(created_at),
        order_count,
        round(total_spend, 2)
    ) AS account_summary
FROM user_metrics
ORDER BY total_spend DESC
LIMIT 10;
```

## Generating SQL Snippets Dynamically

`format()` is useful when you are generating SQL or DSL statements programmatically inside a query, for example when building migration scripts or configuration files from a metadata table:

```sql
-- Generate ALTER TABLE statements for each column needing a type change
SELECT
    format(
        'ALTER TABLE {} MODIFY COLUMN {} {};',
        table_name,
        column_name,
        new_type
    ) AS migration_statement
FROM pending_migrations
WHERE status = 'pending'
ORDER BY table_name, column_name;
```

```sql
-- Generate INSERT statements from a staging table for export
SELECT
    format(
        'INSERT INTO events VALUES ({}, {}, \'{}\', \'{}\');',
        event_id,
        user_id,
        replaceAll(event_type, '\'', '\'\''),
        toDateTime(event_time)
    ) AS insert_statement
FROM events_staging
LIMIT 100;
```

## Building URL or Path Strings

Constructing URLs from individual components is a common use case in analytics pipelines that generate links or canonical identifiers:

```sql
-- Build a canonical URL from base, path, and query components
SELECT
    page_id,
    format('https://{}/{}?ref={}', base_domain, url_path, ref_source) AS canonical_url
FROM page_metadata
LIMIT 10;
```

```sql
-- Compose S3 object paths from partition columns
SELECT
    event_date,
    format(
        's3://my-bucket/events/year={}/month={}/day={}/part-{}.parquet',
        toYear(event_date),
        toMonth(event_date),
        toDayOfMonth(event_date),
        partition_id
    ) AS s3_path
FROM export_partitions
ORDER BY event_date DESC
LIMIT 10;
```

## Combining format() with Aggregation

`format()` works in any expression context, including inside aggregate output:

```sql
-- Report per-service error rate as a formatted percentage string
SELECT
    service,
    format(
        '{} errors / {} requests = {}%',
        countIf(is_error),
        count(),
        round(100.0 * countIf(is_error) / count(), 2)
    ) AS error_rate_summary
FROM requests
WHERE request_date = today() - 1
GROUP BY service
ORDER BY service;
```

```sql
-- Summarize top-5 error types per host as a formatted list
SELECT
    host,
    arrayStringConcat(
        groupArray(5)(
            format('{}: {}', error_type, count)
        ),
        ', '
    ) AS top_errors
FROM (
    SELECT host, error_type, COUNT(*) AS count
    FROM errors
    GROUP BY host, error_type
    ORDER BY host, count DESC
)
GROUP BY host
ORDER BY host
LIMIT 10;
```

## Escaping Braces

To include a literal `{` or `}` in the output, double it:

```sql
SELECT format('JSON example: {{"key": "{}"}}', 'value') AS result;
-- result: 'JSON example: {"key": "value"}'
```

## Comparison with String Concatenation

`format()` is syntactic sugar over string concatenation with automatic type coercion. The two expressions below are equivalent, but `format()` is more readable at scale:

```sql
-- Concatenation style
SELECT toString(user_id) || ' - ' || email || ' - ' || toString(created_at) AS row_label
FROM users LIMIT 5;

-- format() style
SELECT format('{} - {} - {}', user_id, email, created_at) AS row_label
FROM users LIMIT 5;
```

When the template has many placeholders or the surrounding SQL is already complex, `format()` significantly reduces visual noise.

## Summary

`format(template, arg1, arg2, ...)` replaces each `{}` placeholder in the template with the corresponding argument, converting non-string types automatically. It is the cleanest way to compose strings from multiple column values in ClickHouse, outperforming manual concatenation with `||` in readability when more than two or three values are involved. Common uses include formatting alert and log messages, building human-readable report lines, generating SQL or configuration snippets, and constructing URLs or storage paths from partition columns. Use `{{` and `}}` to include literal brace characters in the output.
