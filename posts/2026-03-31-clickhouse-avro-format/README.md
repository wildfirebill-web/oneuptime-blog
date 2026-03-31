# How to Use Avro Format in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Avro, Kafka, Data Engineering

Description: Learn how to read and write Avro files in ClickHouse, including schema registry integration, Kafka ingestion, and type mapping for robust data pipelines.

## What Is Avro?

Apache Avro is a row-based binary serialization format defined by a JSON schema. Unlike Parquet or ORC, Avro embeds its schema in every file, making it self-describing. Avro is the most common format for Kafka messages and is deeply integrated with the Confluent Schema Registry.

ClickHouse supports two Avro variants:
- `Avro` - standard Avro file format (object container file)
- `AvroConfluent` - Avro with a Confluent-compatible schema ID prefix (used for Kafka topics)

## Reading an Avro File

```sql
SELECT *
FROM file('orders.avro', Avro)
LIMIT 10;
```

Inspect the schema embedded in the file:

```sql
DESCRIBE file('orders.avro', Avro);
```

## Loading Avro Data into a Table

```sql
CREATE TABLE orders
(
    order_id    UInt64,
    customer_id UInt32,
    status      LowCardinality(String),
    total       Float64,
    created_at  DateTime
)
ENGINE = MergeTree()
ORDER BY (created_at, customer_id);

INSERT INTO orders
SELECT *
FROM file('orders.avro', Avro);
```

## Writing Data to Avro

Export a query result to an Avro file:

```sql
SELECT order_id, customer_id, total, created_at
FROM orders
WHERE status = 'completed'
INTO OUTFILE 'completed_orders.avro'
FORMAT Avro;
```

From the shell:

```bash
clickhouse-client \
  --query "SELECT * FROM orders FORMAT Avro" \
  > orders_export.avro
```

## AvroConfluent for Kafka Integration

When consuming Avro messages from a Kafka topic that uses Confluent Schema Registry, use `AvroConfluent` and specify the registry URL:

```sql
CREATE TABLE kafka_orders
(
    order_id    UInt64,
    customer_id UInt32,
    total       Float64
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list  = 'orders',
    kafka_group_name  = 'clickhouse_consumer',
    kafka_format      = 'AvroConfluent';
```

Set the schema registry URL in your ClickHouse configuration or as a session setting:

```sql
SET format_avro_schema_registry_url = 'http://schema-registry:8081';
```

## Writing Avro for Kafka Producers

ClickHouse can also produce Avro messages for Kafka:

```sql
INSERT INTO FUNCTION kafka(
    'kafka:9092',
    'order_events',
    '',
    'AvroConfluent'
)
SELECT order_id, customer_id, total
FROM orders
WHERE created_at >= now() - INTERVAL 1 HOUR;
```

## Type Mapping

| ClickHouse Type | Avro Type |
|-----------------|-----------|
| Int8 / UInt8    | int |
| Int32 / UInt32  | int |
| Int64 / UInt64  | long |
| Float32         | float |
| Float64         | double |
| Boolean         | boolean |
| String          | string or bytes |
| UUID            | string (logicalType: uuid) |
| Date            | int (logicalType: date) |
| DateTime        | long (logicalType: timestamp-millis) |
| Array(T)        | array |
| Map(String, V)  | map |
| Nullable(T)     | union [null, T] |

## Schema Evolution

Avro supports schema evolution - readers with a newer schema can read files written with an older schema, provided backward-compatible changes are made (adding fields with defaults, removing fields).

When reading an Avro file with extra fields not present in your table:

```sql
SET input_format_avro_allow_missing_fields = 1;

INSERT INTO orders
SELECT order_id, customer_id, total, created_at
FROM file('orders_v2.avro', Avro);
```

## Generating an Avro Schema from ClickHouse

Use the `avroToClickHouseType` function or generate an Avro schema for your table to share with producers:

```sql
SELECT formatAvroSchemaOneLine(
    ['order_id', 'customer_id', 'total'],
    ['UInt64', 'UInt32', 'Float64']
);
```

## Performance Tips

1. Avro is row-based, so it is better suited for streaming (Kafka) than for bulk analytics.
2. For OLAP queries, prefer Parquet or ORC.
3. When reading large Avro files for bulk import, use `SELECT ... FROM file(...) FORMAT Avro` to let ClickHouse batch the inserts.
4. Compress Avro files with Snappy or Deflate for Kafka; the codec is stored in the file header and ClickHouse handles decompression transparently.

## Conclusion

Avro is the best choice for event-driven architectures where Kafka is the data backbone. Its self-describing schema and first-class Schema Registry support make it reliable for long-running pipelines where the message schema evolves over time. For bulk analytics, combine Avro ingestion with a ClickHouse MergeTree table to get both streaming reliability and analytical speed.

**Related Reading:**

- [How to Use JSONEachRow Format in ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-jsoneachrow-format/view)
- [How to Handle Schema Evolution When Loading Parquet in ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-parquet-schema-evolution/view)
- [How to Import Data from S3 in Various Formats in ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-import-from-s3/view)
