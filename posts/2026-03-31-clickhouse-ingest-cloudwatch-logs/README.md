# How to Ingest CloudWatch Logs into ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, CloudWatch, AWS, Log Ingestion, Lambda

Description: Learn how to forward AWS CloudWatch Logs to ClickHouse using a Lambda subscription filter or the OpenTelemetry Collector for centralized log analytics.

---

CloudWatch Logs is the default log store for AWS services but is expensive to query at scale. Forwarding logs to ClickHouse enables fast, cost-effective analytics.

## Option 1 - Lambda Subscription Filter

Create a Lambda that receives CloudWatch log events and writes to ClickHouse:

```python
import boto3
import base64
import gzip
import json
import os
import urllib.request

CLICKHOUSE_URL = os.environ['CLICKHOUSE_URL']
CLICKHOUSE_USER = os.environ['CLICKHOUSE_USER']
CLICKHOUSE_PASSWORD = os.environ['CLICKHOUSE_PASSWORD']

def handler(event, context):
    payload = base64.b64decode(event['awslogs']['data'])
    log_data = json.loads(gzip.decompress(payload))

    rows = []
    for log_event in log_data['logEvents']:
        rows.append(json.dumps({
            'ts': log_event['timestamp'] // 1000,
            'log_group': log_data['logGroup'],
            'log_stream': log_data['logStream'],
            'message': log_event['message']
        }))

    body = '\n'.join(rows).encode()
    req = urllib.request.Request(
        f"{CLICKHOUSE_URL}?query=INSERT+INTO+cloudwatch_logs+FORMAT+JSONEachRow",
        data=body,
        headers={
            'X-ClickHouse-User': CLICKHOUSE_USER,
            'X-ClickHouse-Key': CLICKHOUSE_PASSWORD
        },
        method='POST'
    )
    urllib.request.urlopen(req)
```

## Target Table

```sql
CREATE TABLE cloudwatch_logs (
    ts DateTime,
    log_group LowCardinality(String),
    log_stream String,
    message String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (log_group, ts)
TTL ts + INTERVAL 90 DAY;
```

## Attaching the Subscription Filter

```bash
aws logs put-subscription-filter \
  --log-group-name /aws/lambda/my-function \
  --filter-name ClickHouseForwarder \
  --filter-pattern "" \
  --destination-arn arn:aws:lambda:us-east-1:123456789:function:clickhouse-forwarder
```

## Option 2 - OpenTelemetry Collector with CloudWatch Receiver

```yaml
receivers:
  awscloudwatch:
    region: us-east-1
    logs:
      poll_interval: 1m
      groups:
        named:
          /aws/lambda/my-function:

exporters:
  clickhouse:
    endpoint: tcp://clickhouse:9000
    database: otel
    logs_table_name: cloudwatch_logs

service:
  pipelines:
    logs:
      receivers: [awscloudwatch]
      exporters: [clickhouse]
```

## Querying the Data

```sql
SELECT
    log_group,
    countIf(message LIKE '%ERROR%') AS error_count,
    count() AS total
FROM cloudwatch_logs
WHERE ts >= now() - INTERVAL 1 HOUR
GROUP BY log_group
ORDER BY error_count DESC;
```

## Summary

Forward CloudWatch Logs to ClickHouse via a Lambda subscription filter for near-real-time delivery, or via the OpenTelemetry Collector CloudWatch receiver for a fully managed pipeline. Query aggregated log analytics at a fraction of CloudWatch Logs Insights cost.
