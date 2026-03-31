# How to Use CSV and CSVWithNames Format in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, CSV Format, CSVWithNames Format, Data Import, Data Export

Description: Learn how to use CSV and CSVWithNames formats in ClickHouse for importing and exporting tabular data with proper delimiter and escaping handling.

---

CSV (Comma-Separated Values) is one of the most widely used data exchange formats. ClickHouse supports CSV and CSVWithNames as first-class input/output formats, allowing you to import data from spreadsheets, export query results for downstream tools, and interface with virtually any data pipeline.

## CSV Format Basics

In ClickHouse's CSV format, each row is a line, fields are separated by commas, and string values are quoted with double quotes when they contain special characters:

```csv
1,Alice,alice@example.com,2024-01-15
2,Bob,"bob,jr@example.com",2024-02-20
3,Charlie,charlie@example.com,2024-03-01
```

## CSVWithNames Format

CSVWithNames adds a header row with column names as the first line:

```csv
id,name,email,created_at
1,Alice,alice@example.com,2024-01-15
2,Bob,bob@example.com,2024-02-20
```

This is the recommended format when sharing CSV files with external tools, as column names are preserved.

## Importing CSV Data

Using clickhouse-client:

```bash
clickhouse-client \
    --query "INSERT INTO users FORMAT CSV" \
    < users.csv
```

With CSVWithNames (header row is automatically skipped):

```bash
clickhouse-client \
    --query "INSERT INTO users FORMAT CSVWithNames" \
    < users_with_header.csv
```

## Importing via HTTP Interface

```bash
curl -X POST \
    "http://localhost:8123/?query=INSERT+INTO+users+FORMAT+CSVWithNames" \
    --data-binary @users.csv
```

## Exporting Query Results as CSV

```sql
SELECT id, name, email, created_at
FROM users
WHERE active = 1
FORMAT CSV;
```

Export to file via clickhouse-client:

```bash
clickhouse-client \
    --query "SELECT * FROM users FORMAT CSVWithNames" \
    > export.csv
```

## Custom Delimiter

The default delimiter is a comma, but you can change it:

```sql
SET format_csv_delimiter = '\t';

SELECT * FROM users FORMAT CSV;
```

This produces tab-separated output while still using the CSV parser rules (quoting, escaping).

## Handling NULL Values

By default, NULL is represented as `\N` in CSV:

```csv
1,Alice,\N,2024-01-15
```

You can customize the NULL representation:

```sql
SET format_csv_null_representation = 'NULL';
SELECT * FROM users FORMAT CSV;
```

## Skipping Rows

When importing, skip a known number of header or metadata rows:

```sql
SET input_format_csv_skip_first_lines = 2;
INSERT INTO users FORMAT CSV;
```

## Schema Inference

ClickHouse can infer the schema from a CSVWithNames file:

```sql
DESCRIBE TABLE file('users.csv', CSVWithNames);
```

## Summary

CSV and CSVWithNames are the go-to formats for interoperability between ClickHouse and external tools. Use CSVWithNames when column names matter for downstream consumers. Customize the delimiter, null representation, and skip settings as needed for your data sources. For bulk imports, the HTTP interface or clickhouse-client pipe is the most efficient path.
