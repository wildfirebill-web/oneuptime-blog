# How to Load Your First Dataset into ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Beginner, Data Loading, CSV, JSONEachRow, Import, Tutorial

Description: Load your first dataset into ClickHouse from CSV, JSON, and Parquet files using clickhouse-client and the HTTP interface with real examples.

---

## Choosing the Right Insert Format

ClickHouse supports many input formats. The most common for beginners:

| Format | Best For |
|--------|---------|
| `CSV` | Spreadsheet exports, simple tabular data |
| `JSONEachRow` | API responses, log files |
| `Parquet` | Data lake files, Spark output |
| `TSV` | Tab-separated data |

## Step 1 - Create Your Table

```sql
CREATE TABLE sales (
    sale_id    UInt64,
    product    LowCardinality(String),
    quantity   UInt32,
    price      Float64,
    sale_date  Date,
    region     LowCardinality(String)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(sale_date)
ORDER BY (region, sale_date, product);
```

## Step 2 - Load from CSV

Create a sample CSV file:

```bash
cat > /tmp/sales.csv << 'EOF'
1,Widget A,10,9.99,2026-01-15,US
2,Widget B,5,24.99,2026-01-15,EU
3,Widget A,20,9.99,2026-01-16,US
4,Widget C,3,49.99,2026-01-16,APAC
EOF
```

Load it:

```bash
clickhouse-client \
  --query="INSERT INTO sales FORMAT CSV" \
  < /tmp/sales.csv
```

With a header row:

```bash
clickhouse-client \
  --query="INSERT INTO sales FORMAT CSVWithNames" \
  < /tmp/sales_with_header.csv
```

## Step 3 - Load from JSONEachRow

```bash
cat > /tmp/sales.jsonl << 'EOF'
{"sale_id":5,"product":"Widget D","quantity":7,"price":14.99,"sale_date":"2026-01-17","region":"US"}
{"sale_id":6,"product":"Widget A","quantity":15,"price":9.99,"sale_date":"2026-01-17","region":"EU"}
EOF

clickhouse-client \
  --query="INSERT INTO sales FORMAT JSONEachRow" \
  < /tmp/sales.jsonl
```

## Step 4 - Load via HTTP

The HTTP interface is useful for scripting and automation:

```bash
curl -X POST 'http://localhost:8123/?query=INSERT+INTO+sales+FORMAT+JSONEachRow' \
  -H 'Content-Type: application/json' \
  --data-binary @/tmp/sales.jsonl
```

## Step 5 - Load from Parquet

```bash
clickhouse-client \
  --query="INSERT INTO sales FORMAT Parquet" \
  < /tmp/sales.parquet
```

Or query Parquet directly without loading:

```sql
SELECT * FROM file('/tmp/sales.parquet', 'Parquet') LIMIT 5;
```

## Step 6 - Verify the Load

```sql
SELECT
    count()          AS total_rows,
    sum(price * quantity) AS total_revenue,
    min(sale_date)   AS first_date,
    max(sale_date)   AS last_date
FROM sales;
```

Check part count and size:

```sql
SELECT
    count()     AS parts,
    sum(rows)   AS rows,
    formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.parts
WHERE table = 'sales' AND active;
```

## Summary

Load your first dataset into ClickHouse by creating a MergeTree table, then using `clickhouse-client` with format flags (CSV, JSONEachRow, Parquet) to stream data from files. Verify the load with count and aggregate queries, and check `system.parts` to confirm data is written correctly.
