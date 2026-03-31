# How to Integrate ClickHouse with AWS Glue

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, AWS Glue, AWS, ETL, Data Catalog, S3, Integration

Description: Integrate ClickHouse with AWS Glue for ETL jobs that move data between ClickHouse and S3, and register ClickHouse tables in the Glue Data Catalog.

---

AWS Glue is a serverless ETL and data catalog service. Integrating ClickHouse with Glue enables you to incorporate ClickHouse into AWS-native data pipelines and manage ClickHouse metadata in the Glue Data Catalog.

## Option 1: Glue ETL Job Writing to ClickHouse

Create a Glue ETL job that reads from S3 and writes to ClickHouse:

```python
import sys
import json
import urllib.request
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'CLICKHOUSE_HOST'])
sc = SparkContext()
glue_context = GlueContext(sc)

# Read from S3 via Glue catalog
datasource = glue_context.create_dynamic_frame.from_catalog(
    database="analytics",
    table_name="events_raw"
)

df = datasource.toDF()

# Collect batches and write to ClickHouse
def write_partition(rows):
    batch = list(rows)
    if not batch:
        return
    body = '\n'.join(json.dumps(row.asDict()) for row in batch)
    url = f"http://{args['CLICKHOUSE_HOST']}:8123/?query=INSERT+INTO+events+FORMAT+JSONEachRow"
    req = urllib.request.Request(url, data=body.encode(), method='POST')
    urllib.request.urlopen(req)

df.foreachPartition(write_partition)
```

## Option 2: Glue Job Reading from ClickHouse via JDBC

Use the ClickHouse JDBC driver in a Glue job to read from ClickHouse:

```python
connection_options = {
    "url": "jdbc:clickhouse://clickhouse-host:8123/analytics",
    "dbtable": "events",
    "user": "default",
    "password": "secret",
    "driver": "com.clickhouse.jdbc.ClickHouseDriver"
}

frame = glue_context.create_dynamic_frame.from_options(
    connection_type="custom.jdbc",
    connection_options=connection_options
)
```

Upload the ClickHouse JDBC jar to S3 and reference it in the Glue job:

```bash
aws s3 cp clickhouse-jdbc-0.6.0.jar s3://my-bucket/glue-jars/
```

## Register ClickHouse Tables in Glue Catalog

Use the Glue Data Catalog to track ClickHouse table schemas:

```python
import boto3

glue = boto3.client('glue')

glue.create_table(
    DatabaseName='clickhouse_analytics',
    TableInput={
        'Name': 'events',
        'TableType': 'EXTERNAL_TABLE',
        'Parameters': {
            'classification': 'clickhouse',
            'connectionName': 'clickhouse-prod'
        },
        'StorageDescriptor': {
            'Columns': [
                {'Name': 'event_time', 'Type': 'timestamp'},
                {'Name': 'event_type', 'Type': 'string'},
                {'Name': 'user_id', 'Type': 'int'},
                {'Name': 'revenue', 'Type': 'decimal(18,4)'}
            ],
            'Location': 'jdbc:clickhouse://clickhouse-host:8123/analytics',
            'InputFormat': 'org.apache.hadoop.mapred.TextInputFormat',
            'OutputFormat': 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
        }
    }
)
```

## Schedule Regular Exports from ClickHouse to S3

Use a Glue Trigger to schedule a job that exports ClickHouse data to S3:

```python
glue_client.create_trigger(
    Name='daily-clickhouse-export',
    Type='SCHEDULED',
    Schedule='cron(0 2 * * ? *)',
    Actions=[{'JobName': 'clickhouse-to-s3-export'}],
    StartOnCreation=True
)
```

## Summary

ClickHouse integrates with AWS Glue via JDBC for reading data and HTTP for writing. Register ClickHouse table schemas in the Glue Data Catalog to enable cross-service discovery, and schedule regular exports to S3 to feed downstream Glue ETL pipelines. Use Spark foreachPartition with batched HTTP inserts for efficient write throughput from Glue jobs.
