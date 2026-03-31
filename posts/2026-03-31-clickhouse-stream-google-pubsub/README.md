# How to Stream Data from Google Pub/Sub to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Google Pub/Sub, Streaming, Data Pipeline, Real-Time Analytics

Description: Learn how to build a real-time streaming pipeline from Google Cloud Pub/Sub into ClickHouse for fast analytical queries on live data.

---

## Overview

Google Cloud Pub/Sub is a fully managed messaging service for asynchronous communication between services. Streaming Pub/Sub messages into ClickHouse lets you perform real-time analytics on event-driven data without batch delays.

This guide covers two main approaches: using the ClickHouse Kafka engine (via a Pub/Sub-to-Kafka bridge) and using a custom consumer written in Python or Go.

## Prerequisites

- A Google Cloud project with Pub/Sub enabled
- A Pub/Sub topic and subscription
- ClickHouse 22.6 or later
- Google Cloud credentials configured locally

## Approach 1: Python Consumer with Bulk Inserts

The simplest approach is a Python consumer that reads messages from a Pub/Sub subscription and writes them to ClickHouse in batches.

Install dependencies:

```bash
pip install google-cloud-pubsub clickhouse-connect
```

Create the target table in ClickHouse:

```sql
CREATE TABLE events (
    event_id    String,
    user_id     UInt64,
    event_type  LowCardinality(String),
    payload     String,
    received_at DateTime DEFAULT now()
) ENGINE = MergeTree()
ORDER BY (event_type, received_at);
```

Write the consumer:

```python
import json
from google.cloud import pubsub_v1
import clickhouse_connect

client = clickhouse_connect.get_client(host='localhost', port=8123)
subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path('my-project', 'my-subscription')

buffer = []
BATCH_SIZE = 500

def callback(message):
    data = json.loads(message.data.decode('utf-8'))
    buffer.append([
        data['event_id'],
        data['user_id'],
        data['event_type'],
        json.dumps(data.get('payload', {})),
    ])
    message.ack()
    if len(buffer) >= BATCH_SIZE:
        flush()

def flush():
    if buffer:
        client.insert('events', buffer,
            column_names=['event_id', 'user_id', 'event_type', 'payload'])
        buffer.clear()

future = subscriber.subscribe(subscription_path, callback=callback)
future.result()
```

## Approach 2: Pub/Sub to Kafka Bridge

If you already use the ClickHouse Kafka engine, route Pub/Sub messages to Kafka using a Dataflow job or a simple connector, then use the native Kafka table engine.

```sql
CREATE TABLE pubsub_kafka_queue (
    event_id   String,
    user_id    UInt64,
    event_type String,
    payload    String
) ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list  = 'pubsub-events',
    kafka_group_name  = 'clickhouse-consumer',
    kafka_format      = 'JSONEachRow';

CREATE MATERIALIZED VIEW pubsub_mv TO events AS
SELECT event_id, user_id, event_type, payload
FROM pubsub_kafka_queue;
```

## Tuning for Throughput

Increase insert block size to reduce merge pressure:

```sql
SET max_insert_block_size = 100000;
SET min_insert_block_size_rows = 50000;
```

Use async inserts to buffer small writes server-side:

```sql
SET async_insert = 1;
SET wait_for_async_insert = 0;
```

## Monitoring Ingestion

Check lag and insert rates:

```sql
SELECT
    table,
    sum(rows_inserted)   AS total_rows,
    sum(bytes_written)   AS total_bytes,
    max(event_time)      AS last_insert
FROM system.part_log
WHERE event_type = 'NewPart'
  AND table = 'events'
GROUP BY table;
```

## Summary

Streaming Google Pub/Sub data into ClickHouse is straightforward using either a batch Python consumer or a Kafka bridge. Batch your inserts, tune block sizes, and use materialized views to land data into optimized tables. For production workloads, prefer the Kafka engine approach for reliability and back-pressure handling.
