# How to Use ClickHouse with Databricks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Databricks, Spark, Delta Lake, Integration, ETL

Description: Connect Databricks to ClickHouse for bidirectional data exchange, combining Databricks Spark processing with ClickHouse analytical queries.

---

Databricks is a unified analytics platform built on Apache Spark. Integrating ClickHouse with Databricks enables you to use Databricks for heavy data processing and ClickHouse for fast analytical queries on the processed results.

## Connect Databricks to ClickHouse via JDBC

Install the ClickHouse JDBC driver on your Databricks cluster, then connect:

```python
clickhouse_url = "jdbc:clickhouse://clickhouse-host:8123/analytics"
clickhouse_props = {
    "driver": "com.clickhouse.jdbc.ClickHouseDriver",
    "user": "default",
    "password": "secret"
}

# Read from ClickHouse into a Databricks DataFrame
df = spark.read.jdbc(
    url=clickhouse_url,
    table="events",
    properties=clickhouse_props
)

df.show(10)
```

## Write Databricks Data to ClickHouse

After processing data in Databricks, write results back to ClickHouse:

```python
result_df = df.groupBy("event_type").agg(
    count("*").alias("event_count"),
    sum("revenue").alias("total_revenue")
)

result_df.write.jdbc(
    url=clickhouse_url,
    table="event_aggregates",
    mode="append",
    properties=clickhouse_props
)
```

For large datasets, increase JDBC batch size:

```python
clickhouse_props["batchsize"] = "50000"
clickhouse_props["numPartitions"] = "8"
```

## Use Delta Lake as an Intermediate Layer

Databricks Delta Lake is the recommended intermediate layer for Databricks-to-ClickHouse pipelines:

```python
# In Databricks: write processed data to Delta Lake
result_df.write.format("delta").mode("overwrite").save("s3://my-bucket/delta/event_aggregates/")
```

Then read from ClickHouse:

```sql
SELECT *
FROM deltaLake('s3://my-bucket/delta/event_aggregates/', 'ACCESS_KEY', 'SECRET_KEY')
LIMIT 100;
```

## Automate with Databricks Jobs

Schedule a Databricks notebook to run ETL and export to ClickHouse:

```python
# Notebook: etl_to_clickhouse.py
from pyspark.sql import functions as F

df = spark.read.format("delta").load("s3://my-bucket/delta/raw_events/")

processed = df.filter(F.col("event_type").isNotNull()) \
    .withColumn("revenue", F.col("revenue").cast("decimal(18,4)")) \
    .select("event_time", "event_type", "user_id", "revenue")

processed.write.jdbc(
    url=clickhouse_url,
    table="events",
    mode="append",
    properties=clickhouse_props
)

print(f"Inserted {processed.count()} rows into ClickHouse")
```

## Install JDBC Driver on Cluster

In the Databricks cluster init script:

```bash
#!/bin/bash
curl -L https://repo1.maven.org/maven2/com/clickhouse/clickhouse-jdbc/0.6.0/clickhouse-jdbc-0.6.0-all.jar \
  -o /databricks/jars/clickhouse-jdbc.jar
```

## Compare: JDBC vs HTTP

For high-throughput writes, use the ClickHouse HTTP API via pandas:

```python
import pandas as pd
import requests, json

pdf = processed.toPandas()
rows = pdf.to_dict(orient='records')
body = '\n'.join(json.dumps(r) for r in rows)

requests.post(
    "http://clickhouse-host:8123/?query=INSERT+INTO+events+FORMAT+JSONEachRow",
    data=body
)
```

## Summary

ClickHouse and Databricks integrate via JDBC for bidirectional data exchange and via Delta Lake as an intermediate storage layer. Use Databricks for complex Spark-based transformations and write results to ClickHouse for fast analytical queries. For large-scale inserts, use the HTTP API with JSON batches instead of JDBC to avoid driver overhead.
