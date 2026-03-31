# How to Ingest Stackdriver Logs into ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Google Cloud Logging, Stackdriver, Log Ingestion, Pub/Sub

Description: Learn how to forward Google Cloud Logging (Stackdriver) logs to ClickHouse via Pub/Sub and the OpenTelemetry Collector or a custom Cloud Function.

---

Google Cloud Logging (formerly Stackdriver) stores logs from GKE, Cloud Run, and other GCP services. Forwarding them to ClickHouse enables long-term retention and analytical queries at lower cost.

## Architecture

```text
Cloud Logging --> Log Sink (Pub/Sub) --> Cloud Function or OTEL Collector --> ClickHouse
```

## Target Table

```sql
CREATE TABLE gcp_logs (
    ts DateTime,
    log_name LowCardinality(String),
    resource_type LowCardinality(String),
    severity LowCardinality(String),
    text_payload String,
    json_payload String,
    labels String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (log_name, ts)
TTL ts + INTERVAL 90 DAY;
```

## Creating a Log Sink to Pub/Sub

```bash
gcloud logging sinks create clickhouse-sink \
  pubsub.googleapis.com/projects/my-project/topics/clickhouse-logs \
  --log-filter='resource.type="gke_container"'

gcloud pubsub topics create clickhouse-logs
```

## Cloud Function Subscriber

```python
import base64
import json
import urllib.request
import os

def forward_to_clickhouse(event, context):
    data = json.loads(base64.b64decode(event['data']))

    row = json.dumps({
        'ts': data.get('timestamp', '')[:19].replace('T', ' '),
        'log_name': data.get('logName', ''),
        'resource_type': data.get('resource', {}).get('type', ''),
        'severity': data.get('severity', 'DEFAULT'),
        'text_payload': data.get('textPayload', ''),
        'json_payload': json.dumps(data.get('jsonPayload', {})),
        'labels': json.dumps(data.get('labels', {}))
    })

    req = urllib.request.Request(
        f"{os.environ['CLICKHOUSE_URL']}?query=INSERT+INTO+gcp_logs+FORMAT+JSONEachRow",
        data=row.encode(),
        headers={'X-ClickHouse-User': 'default', 'X-ClickHouse-Key': os.environ['CH_PASSWORD']},
        method='POST'
    )
    urllib.request.urlopen(req)
```

## Using OTEL Collector with googlecloudpubsub Receiver

```yaml
receivers:
  googlecloudpubsub:
    project: my-project
    subscription: projects/my-project/subscriptions/clickhouse-sub

exporters:
  clickhouse:
    endpoint: tcp://clickhouse:9000
    database: default
    logs_table_name: gcp_logs

service:
  pipelines:
    logs:
      receivers: [googlecloudpubsub]
      exporters: [clickhouse]
```

## Querying GCP Logs

```sql
SELECT
    log_name,
    severity,
    count() AS cnt
FROM gcp_logs
WHERE ts >= now() - INTERVAL 1 HOUR
GROUP BY log_name, severity
ORDER BY cnt DESC;
```

## Summary

Forward Google Cloud Logging to ClickHouse by creating a log sink to Pub/Sub, then consuming with a Cloud Function or OTEL Collector. This approach provides long-term log retention and fast analytical queries at significantly lower cost than Cloud Logging's built-in storage.
