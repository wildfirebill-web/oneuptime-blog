# How to Use One Format for Single-Value Output in ClickHouse

Author: OneUptime Team

Tags: ClickHouse, Format, One, Output, Scripting

Description: Learn how ClickHouse's One format outputs a single raw value without headers or delimiters, ideal for shell scripts and single-scalar queries.

---

The `One` format in ClickHouse outputs exactly one row and one column as a raw string with no headers, no delimiters, and no newline appended. It is the simplest possible output format and is designed for shell scripts, health checks, or any context where you need a bare scalar value from a query.

## How One Differs from Other Formats

```mermaid
graph LR
    A[SELECT count from events] --> B{FORMAT}
    B -- TabSeparated --> C["1923041\n"]
    B -- JSON --> D["{\"data\":[{\"count()\":1923041}], ...}"]
    B -- One --> E["1923041"]
```

`One` produces the raw value with no surrounding syntax.

## Basic Usage

```sql
SELECT count() FROM events FORMAT One;
```

Output (exactly, no newline):

```text
1923041
```

## Shell Script Usage

This is the primary use case for `One`. Capture a scalar from ClickHouse into a shell variable:

```bash
COUNT=$(clickhouse-client --query "SELECT count() FROM events WHERE ts >= today() FORMAT One")
echo "Today's events: $COUNT"
```

Or use it in a conditional:

```bash
PENDING=$(clickhouse-client --query "SELECT count() FROM jobs WHERE status = 'pending' FORMAT One")
if [ "$PENDING" -gt 1000 ]; then
  echo "WARNING: $PENDING pending jobs" >&2
fi
```

## Health Check Endpoint

Use `One` via the HTTP interface for lightweight health checks:

```bash
# Returns "1" if alive
curl -s "http://localhost:8123/?query=SELECT+1+FORMAT+One"
```

The response is exactly the character `1` with no trailing newline, making it easy to parse in monitoring scripts.

## Kubernetes Liveness Probe Example

```yaml
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - "curl -sf 'http://localhost:8123/?query=SELECT+1+FORMAT+One' | grep -q '^1$'"
  initialDelaySeconds: 30
  periodSeconds: 10
```

## What Happens with Multiple Rows or Columns

`One` only returns the first row and first column of the result. If your query returns multiple rows or columns, the extra data is silently dropped. This means you should always write queries that return exactly one scalar when using this format.

```sql
-- Good: guaranteed single value
SELECT max(ts) FROM events FORMAT One;

-- Risky: returns only the first row
SELECT name FROM tables LIMIT 5 FORMAT One;
-- Output: just the first name
```

## Getting the Latest Checkpoint Timestamp in a Script

```bash
#!/bin/bash

LAST_TS=$(clickhouse-client --query "
  SELECT max(ts)
  FROM events
  WHERE ts >= now() - INTERVAL 1 HOUR
  FORMAT One
")

echo "Last event at: $LAST_TS"

# Use in cron or monitoring alert
if [ -z "$LAST_TS" ]; then
  echo "No recent events - possible ingestion failure"
fi
```

## Comparison with TabSeparated and CSV for Scalars

| Format | Output for `SELECT 42` | Trailing newline | Use case |
|---|---|---|---|
| `One` | `42` | No | Shell capture |
| `TabSeparated` | `42\n` | Yes | General text |
| `CSV` | `42\n` | Yes | CSV pipelines |
| `JSON` | `{"data":[{"42":42}],...}\n` | Yes | REST APIs |

The lack of a trailing newline in `One` is intentional and makes the format ideal for capturing into a shell variable or checking with tools like `grep -q`.

## Summary

Use `One` when you need a bare scalar value from ClickHouse in a shell script, health check, or monitoring probe. It produces exactly one value with no formatting overhead. Always ensure your query returns exactly one row and one column to avoid silent truncation.
