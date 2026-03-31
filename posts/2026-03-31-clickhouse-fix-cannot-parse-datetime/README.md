# How to Fix "Cannot parse datetime" in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, DateTime, Error, Parsing, Troubleshooting

Description: Resolve "Cannot parse datetime" errors in ClickHouse by using the right parsing functions and handling non-standard date formats in input data.

---

ClickHouse is strict about datetime format parsing. "Cannot parse datetime" errors occur when input data contains date strings in unexpected formats that do not match the expected `YYYY-MM-DD HH:MM:SS` pattern.

## Understand ClickHouse DateTime Formats

ClickHouse natively expects:
- `Date`: `YYYY-MM-DD`
- `DateTime`: `YYYY-MM-DD HH:MM:SS`
- `DateTime64`: `YYYY-MM-DD HH:MM:SS.ffffff`

Other formats require explicit parsing functions.

## Use parseDateTime for Custom Formats

```sql
-- Parse a US-style date MM/DD/YYYY
SELECT parseDateTime('03/31/2026', '%m/%d/%Y') AS dt;

-- Parse ISO 8601 with T separator
SELECT parseDateTime('2026-03-31T14:30:00', '%Y-%m-%dT%H:%i:%s') AS dt;

-- Parse Unix timestamp string
SELECT fromUnixTimestamp(toInt64('1743417600')) AS dt;
```

## Use Best-Effort Parsing

For mixed-format data, use `parseDateTimeBestEffort`:

```sql
SELECT
    parseDateTimeBestEffort('March 31, 2026 14:30:00') AS dt1,
    parseDateTimeBestEffort('2026-03-31T14:30:00Z') AS dt2,
    parseDateTimeBestEffort('31 Mar 2026') AS dt3;
```

## Handle Parsing Failures Gracefully

Use `parseDateTimeBestEffortOrNull` or `parseDateTimeBestEffortOrZero` to avoid errors:

```sql
SELECT
    parseDateTimeBestEffortOrNull(date_string) AS parsed_date,
    date_string
FROM raw_events
WHERE parseDateTimeBestEffortOrNull(date_string) IS NULL
LIMIT 10;  -- Find unparseable values
```

## Fix INSERT Format Issues

When inserting data via CSV or other formats:

```sql
-- Tell ClickHouse the input format
INSERT INTO events FORMAT CSV
SETTINGS
    date_time_input_format = 'best_effort';
```

Or in `users.xml`:

```xml
<profiles>
  <default>
    <date_time_input_format>best_effort</date_time_input_format>
  </default>
</profiles>
```

## Handle Epoch Milliseconds

If your source sends millisecond timestamps:

```sql
-- Convert milliseconds to DateTime64
SELECT fromUnixTimestamp64Milli(1743417600000) AS dt;

-- Or create a DateTime64 column
CREATE TABLE events (
    event_time DateTime64(3, 'UTC')
) ENGINE = MergeTree() ORDER BY event_time;
```

## Common Format Specifiers

```text
%Y - 4-digit year
%m - 2-digit month (01-12)
%d - 2-digit day (01-31)
%H - hour (00-23)
%i - minute (00-59)
%s - second (00-59)
%e - day without leading zero
%T - time as HH:MM:SS
```

## Summary

"Cannot parse datetime" errors in ClickHouse are fixed by using `parseDateTime` with explicit format strings, `parseDateTimeBestEffort` for mixed formats, or `parseDateTimeBestEffortOrNull` to identify bad values. Set `date_time_input_format = 'best_effort'` for bulk ingestion of data with non-standard datetime strings.
