# How to Run Your First ClickHouse Query

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Beginner, Getting Started, Query, Tutorial, SQL

Description: Run your first ClickHouse query using the CLI client and HTTP interface, understand query output formats, and explore basic aggregation and filtering syntax.

---

## Connecting to ClickHouse

Use the native client:

```bash
clickhouse-client --host localhost --port 9000 --user default
```

Or use the HTTP interface:

```bash
curl 'http://localhost:8123/?query=SELECT+1'
```

## Your First Query

Start with a simple test:

```sql
SELECT 1 + 1 AS result;
```

```text
result
------
2
```

Check the ClickHouse version:

```sql
SELECT version();
```

## Querying a System Table

ClickHouse includes built-in system tables. View all databases:

```sql
SHOW DATABASES;
```

List tables in the default database:

```sql
SHOW TABLES FROM default;
```

Query the system tables for running processes:

```sql
SELECT query_id, user, elapsed, query
FROM system.processes
ORDER BY elapsed DESC;
```

## Generating Data with numbers()

ClickHouse can generate data on the fly:

```sql
SELECT
    number,
    number * number AS square,
    sqrt(number)    AS root
FROM numbers(10);
```

This is useful for testing queries without needing real data.

## Basic Aggregation

```sql
SELECT
    toStartOfHour(now() - INTERVAL number SECOND) AS hour,
    count()                                        AS fake_events
FROM numbers(86400)  -- last 24 hours of seconds
GROUP BY hour
ORDER BY hour DESC
LIMIT 10;
```

## Filtering and Sorting

```sql
SELECT
    number AS n,
    number % 2 AS is_even
FROM numbers(20)
WHERE number % 2 = 0
ORDER BY n DESC;
```

## Using Output Formats

ClickHouse supports many output formats:

```bash
# Pretty table (default in CLI)
clickhouse-client --query "SELECT 1, 2, 3 FORMAT Pretty"

# JSON output
clickhouse-client --query "SELECT 1 AS a, 2 AS b FORMAT JSON"

# CSV
clickhouse-client --query "SELECT 1, 2 FORMAT CSV"
```

## Explaining a Query Plan

Before running a heavy query, check its execution plan:

```sql
EXPLAIN SELECT count() FROM system.query_log WHERE event_date = today();
```

`EXPLAIN PIPELINE` shows the execution pipeline:

```sql
EXPLAIN PIPELINE SELECT count() FROM numbers(1000000);
```

## Reading from a File

Load a CSV file directly:

```bash
clickhouse-client --query="
SELECT *
FROM file('/tmp/data.csv', 'CSV', 'id UInt32, name String, value Float64')
LIMIT 5
"
```

## Summary

Run your first ClickHouse queries using `clickhouse-client` or the HTTP interface. Start with system tables and the `numbers()` generator to explore SQL syntax without needing existing data. Use `EXPLAIN` to understand query plans before running large scans, and leverage the rich format options (JSON, CSV, Pretty) for different output needs.
