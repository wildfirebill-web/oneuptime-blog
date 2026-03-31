# How to Build a Dead Letter Queue for ClickHouse Ingestion

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Dead Letter Queue, Error Handling, Ingestion, Kafka, Data Pipeline

Description: Learn how to build a dead letter queue pattern for ClickHouse ingestion pipelines to capture, inspect, and replay failed records.

---

A dead letter queue (DLQ) captures records that fail during ingestion so they can be inspected, fixed, and replayed without losing data. This is essential for production data pipelines where schema mismatches, malformed data, or transient errors can silently drop records.

## Why You Need a DLQ

```text
- Malformed JSON breaks batch inserts
- Type mismatches cause row-level failures
- Schema drift causes parse errors
- Network timeouts cause partial inserts
```

## Pattern 1: DLQ Table in ClickHouse

Create a dedicated error table alongside your main table:

```sql
CREATE TABLE events_dlq (
    failed_at DateTime DEFAULT now(),
    raw_data String,
    error_message String,
    source String,
    retry_count UInt8 DEFAULT 0
) ENGINE = MergeTree()
ORDER BY failed_at
TTL failed_at + INTERVAL 30 DAY;
```

In your ingestion pipeline, catch insertion errors and write to the DLQ:

```bash
#!/bin/bash
# Insert to main table, redirect failures to DLQ
clickhouse-client --query "INSERT INTO events FORMAT JSONEachRow" < batch.json 2>error.log

if [ $? -ne 0 ]; then
    ERROR_MSG=$(cat error.log)
    clickhouse-client --query "INSERT INTO events_dlq (raw_data, error_message, source) VALUES ('$(cat batch.json)', '$ERROR_MSG', 'pipeline-1')"
fi
```

## Pattern 2: Kafka DLQ Topic

When using Kafka as a source, route failed messages to a DLQ topic:

```sql
-- Main ingestion table
CREATE TABLE kafka_events_main (
    event_id String,
    user_id UInt64,
    timestamp DateTime,
    payload String
) ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'broker:9092',
    kafka_topic_list = 'events',
    kafka_group_name = 'ch-consumer',
    kafka_format = 'JSONEachRow',
    kafka_skip_broken_messages = 1000;

-- DLQ Kafka table for broken messages
CREATE TABLE kafka_events_dlq (
    raw String
) ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'broker:9092',
    kafka_topic_list = 'events-dlq',
    kafka_group_name = 'ch-dlq-consumer',
    kafka_format = 'LineAsString';
```

## Pattern 3: Row-Level Error Capture with input()

Use ClickHouse's `input()` function to validate before inserting:

```sql
INSERT INTO events
SELECT event_id, user_id, timestamp, payload
FROM input('event_id String, user_id UInt64, timestamp DateTime, payload String')
FORMAT JSONEachRow;
```

Invalid rows trigger an exception that your pipeline can catch and redirect to the DLQ.

## Replaying DLQ Records

Once you have fixed the underlying issue, replay from the DLQ:

```sql
-- Inspect failed records
SELECT error_message, count() AS cnt
FROM events_dlq
WHERE failed_at >= now() - INTERVAL 1 DAY
GROUP BY error_message
ORDER BY cnt DESC;

-- Replay after fixing schema
INSERT INTO events (event_id, payload, timestamp)
SELECT
    JSONExtractString(raw_data, 'event_id'),
    JSONExtractString(raw_data, 'payload'),
    now()
FROM events_dlq
WHERE retry_count = 0
  AND failed_at >= now() - INTERVAL 1 DAY;
```

Mark replayed records:

```sql
ALTER TABLE events_dlq UPDATE retry_count = retry_count + 1
WHERE failed_at >= now() - INTERVAL 1 DAY AND retry_count = 0;
```

## Summary

Building a dead letter queue for ClickHouse ingestion protects against silent data loss from malformed records, schema mismatches, and transient errors. By capturing failed records with full context, you can debug issues and replay data after fixes - ensuring no events are permanently lost.
