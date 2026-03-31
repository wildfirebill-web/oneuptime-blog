# How to Sync DynamoDB Data to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, DynamoDB, AWS, CDC, Data Sync, Analytics

Description: Sync AWS DynamoDB tables to ClickHouse using DynamoDB Streams and Lambda or Debezium for real-time analytics on operational data.

---

## Overview

Amazon DynamoDB is a serverless key-value and document database, optimized for single-digit millisecond reads and writes. It lacks support for complex aggregations and analytical queries. Syncing DynamoDB data to ClickHouse provides full SQL analytics on your operational data.

## Architecture Options

```text
Option A: DynamoDB -> DynamoDB Streams -> Lambda -> ClickHouse (real-time)
Option B: DynamoDB -> DynamoDB Streams -> Kinesis -> Kafka -> ClickHouse
Option C: DynamoDB -> S3 Export -> ClickHouse (batch)
```

## Option A: Lambda to ClickHouse (Real-Time)

Enable DynamoDB Streams on your table via AWS Console or CLI:

```bash
aws dynamodb update-table \
  --table-name orders \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_IMAGE
```

Create a Lambda function triggered by the stream:

```python
import json
import urllib.request
import urllib.parse

CLICKHOUSE_HOST = 'clickhouse-host'
CLICKHOUSE_DB   = 'analytics'

def handler(event, context):
    rows = []
    for record in event['Records']:
        if record['eventName'] in ('INSERT', 'MODIFY'):
            img = record['dynamodb'].get('NewImage', {})
            rows.append({
                'order_id':  img.get('order_id', {}).get('S', ''),
                'user_id':   int(img.get('user_id', {}).get('N', 0)),
                'amount':    float(img.get('amount', {}).get('N', 0)),
                'status':    img.get('status', {}).get('S', ''),
            })

    if rows:
        body = '\n'.join(json.dumps(r) for r in rows).encode()
        req = urllib.request.Request(
            f'http://{CLICKHOUSE_HOST}:8123/?query=INSERT+INTO+{CLICKHOUSE_DB}.orders+FORMAT+JSONEachRow',
            data=body,
            headers={'Content-Type': 'application/x-www-form-urlencoded'}
        )
        urllib.request.urlopen(req)
```

Create the target table:

```sql
CREATE TABLE orders (
    order_id  String,
    user_id   UInt64,
    amount    Float64,
    status    LowCardinality(String),
    synced_at DateTime DEFAULT now()
) ENGINE = ReplacingMergeTree(synced_at)
ORDER BY order_id;
```

## Option B: S3 Export (Batch)

Export the full DynamoDB table to S3:

```bash
aws dynamodb export-table-to-point-in-time \
  --table-arn arn:aws:dynamodb:us-east-1:123456789:table/orders \
  --s3-bucket my-exports-bucket \
  --export-format DYNAMODB_JSON
```

Load the exported JSON into ClickHouse:

```sql
INSERT INTO orders
SELECT
    JSONExtractString(Item, 'order_id.S') AS order_id,
    toUInt64(JSONExtractString(Item, 'user_id.N')) AS user_id,
    toFloat64(JSONExtractString(Item, 'amount.N')) AS amount,
    JSONExtractString(Item, 'status.S') AS status
FROM s3(
    's3://my-exports-bucket/AWSDynamoDB/*/data/*.json.gz',
    'KEY', 'SECRET', 'JSONEachRow'
);
```

## Query in ClickHouse

```sql
SELECT
    status,
    count()      AS orders,
    sum(amount)  AS revenue,
    avg(amount)  AS avg_order
FROM orders
WHERE synced_at >= today() - 30
GROUP BY status
ORDER BY revenue DESC;
```

## Handle Deletes

DynamoDB Streams includes DELETE events. Handle them with a ReplacingMergeTree and a soft-delete flag:

```sql
ALTER TABLE orders ADD COLUMN IF NOT EXISTS is_deleted UInt8 DEFAULT 0;
```

In the Lambda, set `is_deleted = 1` for REMOVE events, then filter in queries:

```sql
SELECT * FROM orders
WHERE is_deleted = 0 FINAL;
```

## Summary

Sync DynamoDB data to ClickHouse using DynamoDB Streams with Lambda for real-time replication or S3 Export for bulk historical loads. Use `ReplacingMergeTree` to handle upserts and a soft-delete flag for deletes. ClickHouse then provides full analytical SQL capabilities over your DynamoDB data.
