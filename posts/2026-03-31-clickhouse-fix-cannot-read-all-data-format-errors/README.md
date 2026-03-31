# How to Fix 'Cannot read all data' Format Errors in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Format, Error, Data Ingestion, Troubleshooting

Description: Resolve 'Cannot read all data' format errors in ClickHouse caused by truncated input, encoding issues, or mismatched column counts.

---

The "Cannot read all data" error in ClickHouse occurs during data ingestion when the input stream ends unexpectedly before all expected rows or columns are parsed. This is common with CSV, TSV, JSON, and binary format imports.

## Understand the Error Context

```text
Code: 33. DB::Exception: Cannot read all data.
Bytes read: 1024. Bytes expected: 4096.
```

The error means the parser expected more data than it received.

## Check for Truncated Files

```bash
# Verify file integrity
wc -l data.csv
tail -5 data.csv  # Check if last rows are complete

# For compressed files
gunzip -t data.csv.gz && echo "OK" || echo "CORRUPT"
```

## Fix Column Count Mismatches

The most common cause is a wrong number of columns:

```sql
-- Check table structure
DESCRIBE TABLE my_table;

-- Test with a small sample first
clickhouse-client --query "INSERT INTO my_table FORMAT CSV" \
  --input_format_allow_errors_num=10 \
  --input_format_allow_errors_ratio=0.01 \
  < sample.csv
```

## Handle Extra or Missing Columns

For CSV with extra columns, use `input_format_csv_skip_extra_columns`:

```sql
SET input_format_csv_skip_extra_columns = 1;
INSERT INTO my_table FORMAT CSV;
```

For missing columns with defaults:

```sql
SET input_format_defaults_for_omitted_fields = 1;
```

## Fix Line Ending Issues

Windows-style line endings (`\r\n`) can confuse parsers:

```bash
# Convert to Unix line endings
dos2unix data.csv
```

Or tell ClickHouse to handle them:

```sql
SET input_format_csv_allow_cr_end_of_line = 1;
```

## Debug with JSONEachRow Format

If CSV parsing fails, try converting to `JSONEachRow` for better error messages:

```bash
# Convert CSV to NDJSON for debugging
python3 -c "
import csv, json, sys
reader = csv.DictReader(sys.stdin)
for row in reader:
    print(json.dumps(row))
" < data.csv | head -5
```

## Use Error Tolerance Settings

For bulk imports where some bad rows are acceptable:

```sql
INSERT INTO my_table
SELECT * FROM file('data.csv', CSV)
SETTINGS
    input_format_allow_errors_num = 1000,
    input_format_allow_errors_ratio = 0.05;
```

## Summary

"Cannot read all data" format errors in ClickHouse are caused by truncated input, column count mismatches, or encoding problems. Validate file integrity before import, check column counts match, use `input_format_allow_errors_num` to skip bad rows in bulk loads, and convert Windows line endings. Use `JSONEachRow` as an intermediate format for better error diagnostics.
