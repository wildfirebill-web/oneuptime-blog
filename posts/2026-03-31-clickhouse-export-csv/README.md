# How to Export ClickHouse Data to CSV Files

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Export, CSV, clickhouse-client, Data Export

Description: Learn how to export ClickHouse query results to CSV files using clickhouse-client, the HTTP interface, and automated export scripts.

---

Exporting ClickHouse data to CSV is common for sharing reports, feeding downstream tools, and creating backups of aggregated results.

## Using clickhouse-client

```bash
clickhouse-client \
  --host clickhouse \
  --user default \
  --password secret \
  --query "SELECT * FROM events WHERE date = today()" \
  --format CSVWithNames \
  > events_export.csv
```

`CSVWithNames` adds a header row with column names.

## Using the HTTP Interface

```bash
curl -X GET \
  'http://clickhouse:8123/?query=SELECT+*+FROM+events+WHERE+date%3Dtoday()&default_format=CSVWithNames' \
  -H 'X-ClickHouse-User: default' \
  -H 'X-ClickHouse-Key: secret' \
  -o events_export.csv
```

## Exporting with Custom Delimiters

Use `TSV` for tab-separated values, or `CustomSeparated` for arbitrary delimiters:

```bash
clickhouse-client \
  --query "SELECT user_id, email, created_at FROM users" \
  --format TabSeparatedWithNames \
  > users.tsv
```

## Compressed Export

Pipe directly to gzip:

```bash
clickhouse-client \
  --query "SELECT * FROM events" \
  --format CSVWithNames \
  | gzip > events.csv.gz
```

## Exporting Large Tables in Chunks

For tables too large to export in one query, paginate with LIMIT/OFFSET or use a date range loop:

```bash
for month in 2026-01 2026-02 2026-03; do
  clickhouse-client \
    --query "SELECT * FROM events WHERE toYYYYMM(ts) = replace('${month}', '-', '')" \
    --format CSVWithNames \
    > "events_${month}.csv"
done
```

## Exporting Directly to S3

Use the `s3` table function to write CSV directly to S3:

```sql
INSERT INTO FUNCTION s3(
    'https://s3.amazonaws.com/mybucket/exports/events.csv.gz',
    'AWS_KEY', 'AWS_SECRET',
    'CSVWithNames'
) SELECT * FROM events WHERE ts >= today() - 1;
```

## Summary

Export ClickHouse data to CSV via `clickhouse-client --format CSVWithNames`, the HTTP interface with `default_format=CSVWithNames`, or directly to S3 using the s3 table function. Compress large exports with gzip and paginate with date ranges for tables that exceed memory limits.
