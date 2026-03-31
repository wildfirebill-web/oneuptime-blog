# How to Use Pretty Format and Its Variants in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Pretty Format, PrettyCompact, PrettySpace, Terminal Output

Description: Learn how to use Pretty format and its variants in ClickHouse for beautifully formatted, human-readable tabular query output in the terminal.

---

Pretty format renders ClickHouse query results as a formatted table with Unicode box-drawing characters, column headers, and alignment. It is the default output format when using clickhouse-client interactively. Several variants trade visual detail for compactness, making Pretty formats useful for dashboards, terminals, and human-readable reports.

## Pretty Format (Default)

When you run a query in clickhouse-client, the default output uses Pretty format:

```sql
SELECT
    event_type,
    count() AS cnt,
    sum(value) AS total
FROM events
GROUP BY event_type
LIMIT 5;
```

Output:

```text
+------------+---------+----------+
| event_type |     cnt |    total |
+============+=========+==========+
| page_view  | 4523190 | 45231900 |
| click      | 1823450 | 18234500 |
| purchase   |  234120 |  2341200 |
| signup     |   12340 |   123400 |
| logout     |   98230 |   982300 |
+------------+---------+----------+
```

## PrettyCompact Format

PrettyCompact removes the separator lines between rows, making output more compact:

```sql
SELECT event_type, count() AS cnt
FROM events
GROUP BY event_type
LIMIT 5
FORMAT PrettyCompact;
```

## PrettyCompactMonoBlock

Similar to PrettyCompact but reads all data into memory before rendering, ensuring consistent column widths:

```sql
SELECT * FROM my_table LIMIT 100 FORMAT PrettyCompactMonoBlock;
```

## PrettyNoEscapes

Removes ANSI escape codes (colors, bold). Useful for piping output or logging:

```sql
SELECT * FROM system.processes FORMAT PrettyNoEscapes;
```

## PrettyCompactNoEscapes

Combines PrettyCompact and no ANSI escapes:

```sql
SELECT * FROM events LIMIT 10 FORMAT PrettyCompactNoEscapes;
```

## PrettySpace

Uses spaces instead of Unicode box-drawing characters:

```sql
SELECT event_type, count() FROM events GROUP BY event_type FORMAT PrettySpace;
```

Output:

```text
 event_type   count()
 page_view    4523190
 click        1823450
```

## Controlling Row Limits

Pretty format by default limits output to 10,000 rows to prevent terminal flooding. Change this:

```sql
SET output_format_pretty_max_rows = 50000;
SELECT * FROM events LIMIT 50000 FORMAT Pretty;
```

## Column Width Limits

Long string values are truncated in Pretty format:

```sql
SET output_format_pretty_max_column_pad_width = 30;
SET output_format_pretty_max_value_width = 100;
```

## Using Pretty in Scripts

For automated reports or monitoring scripts, disable ANSI escapes:

```bash
clickhouse-client \
    --query "SELECT * FROM system.parts WHERE active = 1 FORMAT PrettyNoEscapes" \
    | tee report.txt
```

## Summary

Pretty format and its variants make ClickHouse query output human-friendly. Use the default Pretty for interactive sessions, PrettyCompact for denser output, PrettyNoEscapes for scripting and logging, and PrettySpace for minimal table formatting. All Pretty formats are output-only and cannot be used for data import. For production data pipelines, switch to binary or structured formats like Parquet, Arrow, or JSONEachRow.
