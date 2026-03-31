# How to Use Avro and AvroConfluent Formats in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Avro Format, AvroConfluent Format, Schema Registry, Kafka

Description: Learn how to use Avro and AvroConfluent formats in ClickHouse for schema-evolved data exchange with Kafka and Confluent Schema Registry.

---

Apache Avro is a compact binary serialization format with built-in schema support. ClickHouse supports the Avro format for file-based import/export and AvroConfluent for Kafka pipelines using Confluent Schema Registry. Avro is ideal for scenarios requiring strict schema evolution and compact serialization.

## Avro Format

The Avro format embeds the schema in the file header, making files self-describing:

```bash
# Export to Avro
clickhouse-client \
    --query "SELECT id, ts, event_type, user_id FROM events FORMAT Avro" \
    > events.avro
```

```bash
# Import from Avro
clickhouse-client \
    --query "INSERT INTO events FORMAT Avro" \
    < events_from_upstream.avro
```

## Reading Avro Schema

ClickHouse automatically reads the Avro schema embedded in the file. You can inspect the schema:

```bash
python3 -c "
import avro.datafile, avro.io
with open('events.avro', 'rb') as f:
    reader = avro.datafile.DataFileReader(f, avro.io.DatumReader())
    print(reader.get_meta('avro.schema').decode())
"
```

## Type Mapping

ClickHouse maps Avro types as follows:

```text
Avro int      -> Int32
Avro long     -> Int64
Avro float    -> Float32
Avro double   -> Float64
Avro string   -> String
Avro bytes    -> String
Avro boolean  -> UInt8
Avro array    -> Array
Avro null     -> Nullable
```

## AvroConfluent Format

AvroConfluent is used for Kafka messages serialized with Confluent Schema Registry. The schema is stored in the registry, and each message has a magic byte plus schema ID prepended:

```text
[0x00][schema_id (4 bytes)][avro payload]
```

## Configuring AvroConfluent in ClickHouse

Set the Schema Registry URL:

```sql
SET format_avro_schema_registry_url = 'http://schema-registry:8081';
```

## Using AvroConfluent with Kafka Engine

```sql
CREATE TABLE kafka_events (
    id UInt64,
    event_type String,
    ts DateTime,
    user_id UInt32
)
ENGINE = Kafka()
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list = 'events',
    kafka_group_name = 'clickhouse_consumer',
    kafka_format = 'AvroConfluent';
```

ClickHouse automatically fetches the schema from the registry using the schema ID embedded in each message.

## Schema Evolution

Avro supports backward-compatible schema evolution. Adding a new optional field with a default value is safe:

```json
{
    "type": "record",
    "name": "Event",
    "fields": [
        {"name": "id", "type": "long"},
        {"name": "event_type", "type": "string"},
        {"name": "ts", "type": "long"},
        {"name": "platform", "type": ["null", "string"], "default": null}
    ]
}
```

ClickHouse handles the new nullable field gracefully with NULL values for older messages.

## Kafka-to-ClickHouse Pipeline

Full example with materialized view:

```sql
CREATE MATERIALIZED VIEW kafka_events_mv TO events AS
SELECT
    id,
    event_type,
    toDateTime(ts) AS ts,
    user_id
FROM kafka_events;
```

## Summary

Avro format provides compact, self-describing binary serialization with robust schema evolution. Use the standard Avro format for file-based data exchange and AvroConfluent for Kafka-based pipelines integrated with Confluent Schema Registry. ClickHouse handles schema lookup, type mapping, and nullable fields automatically, making it a reliable choice for schema-enforced streaming pipelines.
