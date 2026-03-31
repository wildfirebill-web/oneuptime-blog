# How to Stream Data from Azure Event Hubs to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Azure Event Hubs, Streaming, Kafka, Data Pipeline

Description: Connect Azure Event Hubs to ClickHouse using the Kafka-compatible endpoint for real-time event ingestion and analytics.

---

## Overview

Azure Event Hubs exposes a Kafka-compatible endpoint, which means the ClickHouse Kafka table engine can consume events from Event Hubs without any additional middleware. This guide walks through configuring the connection and building a reliable ingestion pipeline.

## Prerequisites

- An Azure Event Hubs namespace with a hub created
- ClickHouse 22.6 or later
- Event Hubs connection string with `Listen` permissions

## Configure the Kafka Engine

Azure Event Hubs requires SASL authentication over TLS. Create the raw queue table:

```sql
CREATE TABLE event_hubs_queue (
    device_id    String,
    temperature  Float32,
    humidity     Float32,
    ts           UInt64
) ENGINE = Kafka
SETTINGS
    kafka_broker_list    = 'my-namespace.servicebus.windows.net:9093',
    kafka_topic_list     = 'telemetry',
    kafka_group_name     = 'clickhouse-group',
    kafka_format         = 'JSONEachRow',
    kafka_security_protocol            = 'SASL_SSL',
    kafka_sasl_mechanism               = 'PLAIN',
    kafka_sasl_username                = '$ConnectionString',
    kafka_sasl_password                = 'Endpoint=sb://my-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=XXXX';
```

Replace the `SharedAccessKey` value with your actual connection string.

## Create Target Table and Materialized View

```sql
CREATE TABLE telemetry (
    device_id   String,
    temperature Float32,
    humidity    Float32,
    ts          DateTime,
    INDEX idx_device device_id TYPE bloom_filter GRANULARITY 4
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(ts)
ORDER BY (device_id, ts);

CREATE MATERIALIZED VIEW telemetry_mv TO telemetry AS
SELECT
    device_id,
    temperature,
    humidity,
    toDateTime(ts / 1000) AS ts
FROM event_hubs_queue;
```

## Handle Schema Evolution

Add a fallback column for raw JSON when the schema changes:

```sql
ALTER TABLE telemetry ADD COLUMN raw String DEFAULT '';
```

Update the materialized view to capture raw messages:

```sql
DROP VIEW telemetry_mv;

CREATE MATERIALIZED VIEW telemetry_mv TO telemetry AS
SELECT
    device_id,
    temperature,
    humidity,
    toDateTime(ts / 1000) AS ts,
    toString(_raw_message) AS raw
FROM event_hubs_queue;
```

## Monitor Consumer Lag

Check the Kafka consumer group offset via system tables:

```sql
SELECT *
FROM system.kafka_consumers
WHERE database = currentDatabase();
```

For real-time throughput stats:

```sql
SELECT
    count()                  AS inserts,
    sum(rows)                AS total_rows,
    formatReadableSize(sum(bytes_uncompressed)) AS volume
FROM system.part_log
WHERE event_type = 'NewPart'
  AND table = 'telemetry'
  AND event_time > now() - INTERVAL 5 MINUTE;
```

## Scaling with Multiple Partitions

Event Hubs partitions map to Kafka partitions. Add parallel consumers by increasing `kafka_num_consumers`:

```sql
CREATE TABLE event_hubs_queue ( ... )
ENGINE = Kafka
SETTINGS
    kafka_num_consumers = 4,
    kafka_max_block_size = 65536,
    ...;
```

## Summary

Azure Event Hubs integrates natively with ClickHouse through its Kafka endpoint. Use the Kafka table engine with SASL_SSL credentials, pair it with a materialized view, and tune `kafka_num_consumers` to match your partition count for maximum throughput.
