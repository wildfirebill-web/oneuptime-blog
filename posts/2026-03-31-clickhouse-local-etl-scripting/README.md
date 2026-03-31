# How to Use clickhouse-local for ETL Scripting

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, clickhouse-local, ETL, Data Pipeline, Shell Script

Description: Learn how to use clickhouse-local as an ETL engine in shell scripts to extract, transform, and load data between files and databases.

---

## clickhouse-local as an ETL Tool

`clickhouse-local` handles all three ETL stages: extraction from files or stdin, SQL-based transformation, and loading to files or a remote ClickHouse server. This makes it a lightweight alternative to Python-based ETL scripts for many data pipeline tasks.

## Basic ETL Pipeline Structure

```bash
#!/bin/bash

INPUT_FILE="/data/raw/orders_$(date +%Y%m%d).csv"
OUTPUT_FILE="/data/processed/orders_$(date +%Y%m%d).parquet"

clickhouse local \
  --query "
  SELECT
      order_id,
      trim(customer_name) AS customer_name,
      toFloat64OrZero(amount_str) AS amount,
      parseDateTimeBestEffort(date_str) AS order_date,
      upper(status) AS status
  FROM file('$INPUT_FILE', CSV, 'order_id UInt64, customer_name String, amount_str String, date_str String, status String')
  WHERE toFloat64OrZero(amount_str) > 0
    AND date_str != ''
  " \
  --format Parquet > "$OUTPUT_FILE"

echo "ETL complete: $(wc -l < $INPUT_FILE) rows processed"
```

## Loading to Remote ClickHouse

Extract from a file, transform, and INSERT into a remote server:

```bash
clickhouse local --query "
SELECT
    user_id,
    toDate(event_time) AS event_date,
    event_type,
    sum(amount) AS daily_total
FROM file('events.csv', CSVWithNames)
GROUP BY user_id, event_date, event_type
" --format TSV | clickhouse-client \
    --host prod.clickhouse.local \
    --query "INSERT INTO daily_summaries FORMAT TSV"
```

## Incremental ETL with Date Filtering

```bash
#!/bin/bash
YESTERDAY=$(date -d yesterday +%Y-%m-%d)

clickhouse local --query "
SELECT *
FROM file('/data/events_*.parquet', Parquet)
WHERE toDate(event_time) = '$YESTERDAY'
" --format Parquet > "/data/daily/${YESTERDAY}.parquet"
```

## Multi-Step Transformation Pipeline

```bash
#!/bin/bash

# Step 1: Extract and normalize
clickhouse local \
  --query "SELECT *, lower(trim(email)) AS normalized_email FROM file('raw.csv', CSVWithNames)" \
  --format Parquet > /tmp/step1.parquet

# Step 2: Deduplicate
clickhouse local \
  --query "SELECT DISTINCT * FROM file('/tmp/step1.parquet', Parquet)" \
  --format Parquet > /tmp/step2.parquet

# Step 3: Enrich and output
clickhouse local \
  --query "
  SELECT
      u.*,
      c.country_name,
      c.timezone
  FROM file('/tmp/step2.parquet', Parquet) u
  LEFT JOIN file('country_codes.csv', CSVWithNames) c ON u.country_code = c.code
  " \
  --format CSVWithNames > /data/output/enriched_users.csv

rm /tmp/step1.parquet /tmp/step2.parquet
```

## Data Validation in ETL

```bash
INVALID_COUNT=$(clickhouse local --query "
SELECT count()
FROM file('orders.csv', CSVWithNames)
WHERE amount <= 0 OR customer_id = 0 OR order_date < '2020-01-01'
" --format TabSeparated)

if [ "$INVALID_COUNT" -gt 0 ]; then
  echo "Validation failed: $INVALID_COUNT invalid rows found"
  exit 1
fi
```

## Summary

`clickhouse-local` implements ETL pipelines through shell scripts: extract from files with glob patterns, transform using full ClickHouse SQL, and load to files or remote servers via `clickhouse-client`. Multi-step pipelines chain commands with intermediate Parquet files for efficiency.
