# How to Sync Cassandra Data to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Cassandra, Data Sync, CDC, Analytics

Description: Sync Apache Cassandra tables to ClickHouse for analytical queries using Debezium CDC or batch export with COPY and clickhouse-client.

---

## Overview

Apache Cassandra is designed for high-throughput, low-latency writes at scale, but its query model is limited to primary key lookups and simple scans. Syncing data to ClickHouse unlocks complex SQL analytics, aggregations, and time-series queries on Cassandra data.

## Approach 1: Batch Export with COPY

Export Cassandra data using cqlsh's COPY command:

```bash
cqlsh -e "COPY analytics.events (event_id, user_id, event_type, amount, ts)
           TO '/tmp/events.csv'
           WITH HEADER = TRUE AND DELIMITER = ',';"
```

Load into ClickHouse:

```bash
clickhouse-client \
  --query="INSERT INTO events FORMAT CSVWithNames" \
  < /tmp/events.csv
```

Create the target table:

```sql
CREATE TABLE events (
    event_id   UUID,
    user_id    UUID,
    event_type LowCardinality(String),
    amount     Float64,
    ts         DateTime
) ENGINE = MergeTree()
ORDER BY (event_type, ts);
```

## Approach 2: Debezium Cassandra CDC

Debezium provides a Cassandra CDC connector that reads from commit logs:

```bash
curl -X POST http://kafka-connect:8083/connectors \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "cassandra-source",
    "config": {
      "connector.class": "io.debezium.connector.cassandra.CassandraConnector",
      "cassandra.config": "/etc/cassandra/cassandra.yaml",
      "cassandra.hosts": "cassandra-host",
      "cassandra.port": "9042",
      "cassandra.username": "cassandra",
      "cassandra.password": "cassandra",
      "offset.backing.store.dir": "/tmp/offsets",
      "schema.include.list": "analytics",
      "table.include.list": "analytics.events",
      "topic.prefix": "cassandra"
    }
  }'
```

Consume in ClickHouse:

```sql
CREATE TABLE cassandra_events_queue (
    event_id   String,
    user_id    String,
    event_type String,
    amount     Float64,
    ts         Int64
) ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list  = 'cassandra.analytics.events',
    kafka_group_name  = 'clickhouse-cassandra',
    kafka_format      = 'JSONEachRow';

CREATE MATERIALIZED VIEW cassandra_events_mv TO events AS
SELECT
    toUUID(event_id)          AS event_id,
    toUUID(user_id)           AS user_id,
    event_type,
    amount,
    toDateTime(ts / 1000)     AS ts
FROM cassandra_events_queue;
```

## Approach 3: Python with cassandra-driver

Sync data programmatically with full control:

```python
from cassandra.cluster import Cluster
import clickhouse_connect

cluster = Cluster(['cassandra-host'])
session = cluster.connect('analytics')

ch = clickhouse_connect.get_client(host='localhost')

rows = session.execute(
    "SELECT event_id, user_id, event_type, amount, ts "
    "FROM events WHERE ts > ? ALLOW FILTERING",
    [last_sync_ts]
)

batch = [[str(r.event_id), str(r.user_id), r.event_type,
          float(r.amount or 0), r.ts] for r in rows]

if batch:
    ch.insert('events', batch,
        column_names=['event_id', 'user_id', 'event_type', 'amount', 'ts'])
```

## Analytical Queries in ClickHouse

Once synced, run queries that would be impossible in Cassandra:

```sql
SELECT
    event_type,
    toStartOfHour(ts)  AS hour,
    count()            AS events,
    sum(amount)        AS total
FROM events
WHERE ts >= today() - 7
GROUP BY event_type, hour
ORDER BY hour, total DESC;
```

## Summary

Cassandra is a write-optimized store for operational data; ClickHouse is an analytical engine for complex queries. Use batch COPY exports for simplicity or Debezium CDC for real-time replication. Once data is in ClickHouse, you gain full SQL expressiveness including aggregations, window functions, and joins.
