# How to Fix 'Cannot read all data' Format Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Troubleshooting, Format, Data Ingestion, Parsing

Description: Fix 'Cannot read all data' format errors in ClickHouse caused by malformed input data, incorrect format specification, or corrupted data parts.

---

The "Cannot read all data" error in ClickHouse appears during data ingestion or query execution when the data format does not match the expected structure, or when a data part is corrupted. This guide covers both ingestion and storage causes.

## Error Messages

You may see variations of:

```text
Code: 32. DB::Exception: Cannot read all data. Bytes read: X. Bytes expected: Y.
Code: 27. DB::Exception: Cannot parse input: expected X before Y.
```

## Cause 1: Wrong Format Specification

The most common cause - you are specifying the wrong input format:

```bash
# Wrong: specifying CSV when data is TSV
cat data.tsv | clickhouse-client --query "INSERT INTO events FORMAT CSV"

# Correct: use TabSeparated for TSV
cat data.tsv | clickhouse-client --query "INSERT INTO events FORMAT TabSeparated"
```

Check the correct format:

```sql
-- Test with a small sample first
SELECT * FROM file('/tmp/sample.csv', CSV)
LIMIT 5;
```

## Cause 2: Column Count Mismatch

The input has a different number of columns than the table:

```sql
-- Check table structure
DESCRIBE TABLE mydb.events;
```

If the input has extra or missing columns, specify the columns explicitly:

```bash
clickhouse-client --query "INSERT INTO events (ts, user_id, action) FORMAT CSV" < data.csv
```

## Cause 3: Data Type Mismatch

A field contains a value that cannot be parsed as the target type:

```sql
-- Test parse errors with input_format_allow_errors_num
SET input_format_allow_errors_num = 10;
SET input_format_allow_errors_ratio = 0.1;

INSERT INTO events FORMAT CSV;
-- Paste sample data to find the offending row
```

Find the problematic rows:

```bash
# Check the first row that causes issues
head -100 data.csv | clickhouse-client --query "INSERT INTO events FORMAT CSV" 2>&1
```

## Cause 4: Corrupted Data Part

For "Cannot read all data" during SELECT queries (not inserts), a data part may be corrupted:

```sql
-- Check for corrupted parts
CHECK TABLE mydb.my_table;
```

If corrupted parts are found:

```bash
# On MergeTree tables, mark the corrupted part as detached
# First identify it from the error message, then:
ALTER TABLE mydb.my_table DETACH PART 'corrupted_part_name';
```

The detached part can be deleted or recovered from a backup.

## Cause 5: Incomplete File Upload

If uploading via HTTP and the connection drops mid-upload:

```bash
# Use retry logic
for i in {1..3}; do
  curl -X POST 'http://localhost:8123/?query=INSERT+INTO+events+FORMAT+CSV' \
    --data-binary @data.csv && break
  echo "Retry $i failed"
  sleep 5
done
```

## Validating Data Format

Before inserting large datasets, validate the format:

```sql
-- Use file() table function to inspect without inserting
SELECT count(), min(event_time), max(event_time)
FROM file('/tmp/data.csv', CSV, 'event_time DateTime, user_id UInt64, action String');
```

## Summary

"Cannot read all data" format errors are caused by mismatched format specifications, column count mismatches, data type incompatibilities, or corrupted storage parts. Fix inserts by specifying the correct FORMAT, explicitly listing target columns, and validating data with `file()` or limited inserts before bulk loading. For corrupted parts, use `CHECK TABLE` and `DETACH PART` to isolate and remove the corruption.
