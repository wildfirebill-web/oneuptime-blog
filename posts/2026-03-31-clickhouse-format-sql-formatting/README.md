# How to Use clickhouse-format for SQL Formatting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, clickhouse-format, SQL Formatting, Developer Tool, Linting

Description: Learn how to use clickhouse-format to auto-format ClickHouse SQL queries for consistent style, better readability, and CI linting pipelines.

---

`clickhouse-format` is a command-line tool that parses and reformats ClickHouse SQL queries into a canonical style. It helps enforce consistent SQL formatting across teams and integrates well into CI pipelines.

## Installation

Included with ClickHouse:

```bash
which clickhouse-format
# /usr/bin/clickhouse-format
```

## Basic Formatting

Pipe a query into `clickhouse-format`:

```bash
echo "select user_id,count() from events where ts>now()-interval 1 day group by user_id" \
  | clickhouse-format
```

Output:

```sql
SELECT
    user_id,
    count()
FROM events
WHERE ts > (now() - INTERVAL 1 DAY)
GROUP BY user_id
```

## Formatting a File

```bash
clickhouse-format --query "$(cat my_query.sql)"
```

Or use a shell pipeline:

```bash
clickhouse-format < my_query.sql > formatted.sql
```

## In-Place Formatting for Multiple Files

```bash
for f in queries/*.sql; do
  clickhouse-format < "$f" > "$f.tmp" && mv "$f.tmp" "$f"
done
```

## Highlighting Mode

Output with ANSI syntax highlighting for terminal review:

```bash
clickhouse-format --hilite < my_query.sql
```

## Multiquery Mode

Format multiple queries separated by semicolons:

```bash
clickhouse-format --multiquery < migrations.sql
```

## CI Pipeline Integration

Add a formatting check to your CI:

```bash
#!/usr/bin/env bash
# check-sql-format.sh
for f in queries/*.sql; do
  formatted=$(clickhouse-format < "$f")
  original=$(cat "$f")
  if [ "$formatted" != "$original" ]; then
    echo "FAIL: $f is not formatted correctly"
    exit 1
  fi
done
echo "All SQL files are properly formatted"
```

## Pre-commit Hook

```bash
# .git/hooks/pre-commit
files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.sql$')
for f in $files; do
  clickhouse-format < "$f" > /tmp/formatted.sql
  if ! diff -q "$f" /tmp/formatted.sql > /dev/null; then
    echo "SQL not formatted: $f"
    exit 1
  fi
done
```

## Summary

`clickhouse-format` brings consistent, readable SQL formatting to your ClickHouse development workflow. Integrate it into CI and pre-commit hooks to maintain style standards across your team automatically.
