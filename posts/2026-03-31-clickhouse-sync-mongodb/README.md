# How to Sync MongoDB Data to ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, MongoDB, CDC, Data Sync, ETL

Description: Sync MongoDB collections to ClickHouse using change streams or batch export for real-time and historical analytics on document data.

---

## Overview

MongoDB stores data as BSON documents, while ClickHouse is a columnar analytical database. Syncing data between them enables fast analytics on MongoDB operational data. There are two main approaches: periodic batch export and real-time change stream replication.

## Approach 1: Batch Export via mongoexport

Export a MongoDB collection to JSON and load it into ClickHouse:

```bash
mongoexport \
  --host mongodb://localhost:27017 \
  --db ecommerce \
  --collection orders \
  --type json \
  --out /tmp/orders.jsonl
```

Load into ClickHouse:

```bash
clickhouse-client \
  --query="INSERT INTO orders FORMAT JSONEachRow" \
  < /tmp/orders.jsonl
```

First create the target table:

```sql
CREATE TABLE orders (
    _id       String,
    user_id   String,
    amount    Float64,
    status    LowCardinality(String),
    items     String,
    order_ts  DateTime
) ENGINE = MergeTree()
ORDER BY (user_id, order_ts);
```

## Approach 2: Real-Time with Debezium MongoDB Connector

For live replication, use the Debezium MongoDB source connector:

```bash
curl -X POST http://kafka-connect:8083/connectors \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "mongo-source",
    "config": {
      "connector.class": "io.debezium.connector.mongodb.MongoDbConnector",
      "mongodb.connection.string": "mongodb://reader:pass@mongo:27017",
      "collection.include.list": "ecommerce.orders",
      "topic.prefix": "mongo"
    }
  }'
```

Consume the topic in ClickHouse:

```sql
CREATE TABLE mongo_orders_queue (
    after  String,
    op     LowCardinality(String)
) ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list  = 'mongo.ecommerce.orders',
    kafka_group_name  = 'clickhouse-mongo',
    kafka_format      = 'JSONEachRow';

CREATE MATERIALIZED VIEW mongo_orders_mv TO orders AS
SELECT
    JSONExtractString(after, '_id.$oid')  AS _id,
    JSONExtractString(after, 'user_id')   AS user_id,
    JSONExtractFloat(after, 'amount')     AS amount,
    JSONExtractString(after, 'status')    AS status,
    JSONExtractString(after, 'items')     AS items,
    toDateTime(JSONExtractInt(after, 'order_ts.$date') / 1000) AS order_ts
FROM mongo_orders_queue
WHERE op IN ('c', 'u');
```

## Approach 3: Python Change Stream Consumer

Use PyMongo's change stream API for a lightweight consumer:

```python
import pymongo
import clickhouse_connect
import json

mongo = pymongo.MongoClient('mongodb://localhost:27017')
ch = clickhouse_connect.get_client(host='localhost')

collection = mongo['ecommerce']['orders']

with collection.watch(full_document='updateLookup') as stream:
    for change in stream:
        doc = change.get('fullDocument', {})
        if doc:
            ch.insert('orders', [[
                str(doc.get('_id')),
                str(doc.get('user_id')),
                float(doc.get('amount', 0)),
                doc.get('status', ''),
                json.dumps(doc.get('items', [])),
                doc.get('order_ts'),
            ]], column_names=['_id', 'user_id', 'amount', 'status', 'items', 'order_ts'])
```

## Query Synced Data

```sql
SELECT
    status,
    count()      AS orders,
    sum(amount)  AS revenue
FROM orders
WHERE order_ts >= today() - 30
GROUP BY status
ORDER BY revenue DESC;
```

## Summary

Sync MongoDB data to ClickHouse using batch export for simplicity or Debezium change streams for real-time replication. ClickHouse's JSON extraction functions make it easy to parse MongoDB's nested document structure into typed columns during the transformation step.
