# How to Use format() for Dynamic Query Building in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, format(), Dynamic Query, SQL, String Formatting, Query Building

Description: Learn how to use ClickHouse's format() function to dynamically construct query strings and format output for display and debugging.

---

The `format()` function in ClickHouse is a string formatting utility that supports Python-style positional placeholders. It is useful for constructing dynamic strings, generating labels, and building parameterized SQL snippets.

## Basic format() Syntax

```sql
-- Positional arguments with {0}, {1}, etc.
SELECT format('Hello, {}! You have {} messages.', 'Alice', 42);
-- Output: Hello, Alice! You have 42 messages.

-- Numbered placeholders
SELECT format('{0} + {1} = {2}', 1, 2, 3);
-- Output: 1 + 2 = 3
```

## Dynamic Labels in Query Results

Use `format()` to generate readable labels from multiple columns:

```sql
SELECT
    service,
    endpoint,
    format('{} {}', method, endpoint) AS full_route,
    count() AS requests
FROM http_requests
WHERE timestamp >= now() - INTERVAL 1 HOUR
GROUP BY service, endpoint, method
ORDER BY requests DESC;
```

## Constructing URL-Style Paths

```sql
SELECT
    format('/api/v{}/users/{}/events', api_version, user_id) AS api_path,
    count() AS calls
FROM api_logs
GROUP BY api_version, user_id
ORDER BY calls DESC
LIMIT 20;
```

## Generating Human-Readable Time Ranges

```sql
SELECT
    toStartOfHour(timestamp) AS hour,
    count() AS events,
    format('{} to {}',
        formatDateTime(toStartOfHour(timestamp), '%H:%M'),
        formatDateTime(toStartOfHour(timestamp) + INTERVAL 1 HOUR, '%H:%M')
    ) AS time_range
FROM events
WHERE timestamp >= today()
GROUP BY hour
ORDER BY hour;
```

## Dynamic Error Messages in Transforms

```sql
INSERT INTO events_clean
SELECT
    event_id,
    user_id,
    if(
        length(event_type) = 0,
        'unknown',
        event_type
    ) AS event_type,
    timestamp
FROM events_raw;

-- Log transformation issues
SELECT
    event_id,
    format('Field {} is invalid: got {} (expected {})',
        'user_id',
        toString(user_id),
        'positive integer'
    ) AS error_message
FROM events_raw
WHERE user_id <= 0;
```

## Building S3 Paths Dynamically

```sql
SELECT
    format('s3://my-bucket/{}/{}/{}/events.parquet',
        toYear(day),
        leftPad(toString(toMonth(day)), 2, '0'),
        leftPad(toString(toDayOfMonth(day)), 2, '0')
    ) AS s3_path,
    count() AS event_count
FROM (
    SELECT toDate(timestamp) AS day
    FROM events
    GROUP BY day
);
```

## Using format() with SETTINGS

Note that ClickHouse also has a `FORMAT` keyword for output format selection, which is different from the `format()` function:

```sql
-- This is the FORMAT keyword (output format), not format() function
SELECT * FROM events FORMAT JSON;
SELECT * FROM events FORMAT Parquet;
```

## Summary

The `format()` function in ClickHouse is a versatile string formatter that supports positional placeholders. It is useful for generating human-readable labels, constructing dynamic paths, creating diagnostic messages, and enriching query output without separate string concatenation. Be careful not to confuse it with the `FORMAT` output specifier keyword.
