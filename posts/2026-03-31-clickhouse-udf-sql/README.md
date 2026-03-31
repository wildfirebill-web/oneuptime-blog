# How to Create UDFs in ClickHouse with SQL

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, UDF, SQL Function, User-Defined Function, Query

Description: Learn how to create SQL-based user-defined functions in ClickHouse to encapsulate reusable expressions and simplify complex query logic.

---

ClickHouse supports user-defined functions (UDFs) written in SQL. SQL UDFs let you encapsulate reusable expressions, complex calculations, or formatting logic into named functions that can be called from any query.

## Creating a SQL UDF

Use `CREATE FUNCTION` with a lambda expression:

```sql
CREATE FUNCTION formatBytes AS (n) ->
    multiIf(
        n >= 1073741824, concat(toString(round(n / 1073741824, 2)), ' GB'),
        n >= 1048576,    concat(toString(round(n / 1048576,    2)), ' MB'),
        n >= 1024,       concat(toString(round(n / 1024,       2)), ' KB'),
        concat(toString(n), ' B')
    );
```

## Calling the UDF

```sql
SELECT
    request_id,
    formatBytes(response_bytes) AS human_size
FROM access_logs
LIMIT 10;
```

## Multi-Argument UDF

```sql
CREATE FUNCTION calcTax AS (amount, rate) ->
    round(amount * rate, 2);
```

```sql
SELECT calcTax(100.00, 0.085) AS tax_amount;
-- Returns: 8.5
```

## UDF with Date Logic

```sql
CREATE FUNCTION isBusiness AS (dt) ->
    toDayOfWeek(dt) BETWEEN 1 AND 5;
```

```sql
SELECT
    event_id,
    event_time,
    isBusiness(event_time) AS is_business_hours
FROM events
LIMIT 10;
```

## Listing Existing UDFs

```sql
SELECT name, create_query
FROM system.functions
WHERE origin = 'SQLUserDefined';
```

## Dropping a UDF

```sql
DROP FUNCTION formatBytes;
```

## Replacing a UDF

```sql
CREATE OR REPLACE FUNCTION formatBytes AS (n) ->
    multiIf(
        n >= 1099511627776, concat(toString(round(n / 1099511627776, 2)), ' TB'),
        n >= 1073741824,    concat(toString(round(n / 1073741824,    2)), ' GB'),
        n >= 1048576,       concat(toString(round(n / 1048576,       2)), ' MB'),
        n >= 1024,          concat(toString(round(n / 1024,          2)), ' KB'),
        concat(toString(n), ' B')
    );
```

## Practical Example - Clean URL Normalization

```sql
CREATE FUNCTION normalizeUrl AS (url) ->
    lower(replaceRegexpOne(url, '^https?://(www\\.)?', ''));
```

```sql
SELECT
    normalizeUrl(referrer) AS clean_referrer,
    count() AS visits
FROM page_views
GROUP BY clean_referrer
ORDER BY visits DESC
LIMIT 10;
```

## Limitations

- SQL UDFs do not support aggregate operations (use SQL aggregate UDFs for that)
- UDFs are stored on the server and persist across restarts
- UDF names are global per database instance

## Summary

SQL UDFs in ClickHouse let you extract repeated expressions into named functions using `CREATE FUNCTION`. They integrate seamlessly into SELECT, WHERE, and GROUP BY clauses, making queries shorter and easier to maintain without any performance penalty compared to inline expressions.
