# How to Export ClickHouse Data to Google Cloud Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Google Cloud Storage, GCS, Export, Data Lake

Description: Learn how to export ClickHouse query results to Google Cloud Storage using the GCS table function in CSV, Parquet, and JSON formats.

---

ClickHouse supports writing directly to Google Cloud Storage (GCS) using the `gcs` table function, enabling seamless data lake integration with BigQuery, Dataflow, and Spark.

## Basic GCS Export

```sql
INSERT INTO FUNCTION gcs(
    'https://storage.googleapis.com/my-bucket/exports/events.csv',
    'HMAC_ACCESS_KEY',
    'HMAC_SECRET',
    'CSVWithNames'
)
SELECT * FROM events
WHERE toDate(ts) = today() - 1;
```

Generate HMAC keys from the GCS console under Settings - Interoperability.

## Exporting Parquet to GCS

```sql
INSERT INTO FUNCTION gcs(
    'https://storage.googleapis.com/my-data-lake/events/date=2026-03-31/data.parquet',
    'HMAC_KEY', 'HMAC_SECRET',
    'Parquet'
)
SELECT * FROM events
WHERE toDate(ts) = '2026-03-31';
```

Parquet files in GCS with Hive-style partitioning are directly queryable from BigQuery:

```sql
-- In BigQuery
CREATE EXTERNAL TABLE my_dataset.events
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://my-data-lake/events/date=*/*.parquet'],
  hive_partition_uri_prefix = 'gs://my-data-lake/events/'
);
```

## Using GCS with a Service Account

Configure a service account JSON key in `config.xml`:

```xml
<gcs>
  <endpoint>https://storage.googleapis.com/my-bucket/</endpoint>
  <access_key_id>HMAC_ACCESS_KEY</access_key_id>
  <secret_access_key>HMAC_SECRET</secret_access_key>
</gcs>
```

## Exporting JSON Lines to GCS

```sql
INSERT INTO FUNCTION gcs(
    'https://storage.googleapis.com/my-bucket/events/2026-03-31.ndjson.gz',
    'HMAC_KEY', 'HMAC_SECRET',
    'JSONEachRow',
    'gzip'
)
SELECT * FROM events
WHERE toDate(ts) = '2026-03-31';
```

## Automated Export Script

```bash
#!/bin/bash
DATE=$(date -d yesterday +%Y-%m-%d)
clickhouse-client \
  --query "
    INSERT INTO FUNCTION gcs(
      'https://storage.googleapis.com/my-bucket/events/date=${DATE}/data.parquet',
      '${GCS_HMAC_KEY}', '${GCS_HMAC_SECRET}',
      'Parquet'
    )
    SELECT * FROM events WHERE toDate(ts) = '${DATE}'
  "
```

## Verifying the Export

```sql
SELECT count()
FROM gcs(
    'https://storage.googleapis.com/my-bucket/exports/events.csv',
    'HMAC_KEY', 'HMAC_SECRET',
    'CSVWithNames'
);
```

## Summary

ClickHouse exports to GCS using HMAC keys via the `gcs` table function. Write Parquet with Hive-style date partitioning for BigQuery external table compatibility. Automate daily exports with a scheduled script and compress large JSON exports with gzip.
