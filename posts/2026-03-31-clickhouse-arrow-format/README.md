# How to Use Arrow Format in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Arrow, Data Engineering, Analytics

Description: Learn how to use Apache Arrow IPC format in ClickHouse for high-speed zero-copy data exchange with Python, Spark, and other Arrow-native systems.

## What Is Arrow Format?

Apache Arrow defines an in-memory columnar data format and a serialization protocol called Arrow IPC (Inter-Process Communication). The Arrow IPC file format (also called Feather v2) is a stream of record batches with a schema header. Because Arrow is designed for in-memory processing, transferring data between Arrow-native systems often requires no serialization at all - the bytes can be used directly.

ClickHouse supports two Arrow variants:
- `Arrow` - the Arrow IPC file format (random access)
- `ArrowStream` - the Arrow IPC stream format (sequential, no seeking required)

Arrow is ideal when your Python (PyArrow, Pandas, Polars) or JVM (Arrow Flight) code needs to exchange data with ClickHouse at high throughput.

## Querying an Arrow File Directly

```sql
SELECT *
FROM file('metrics.arrow', Arrow)
LIMIT 5;
```

Inspect the schema:

```sql
DESCRIBE file('metrics.arrow', Arrow);
```

## Loading Arrow Data into a Table

```sql
CREATE TABLE metrics
(
    metric_id   UInt64,
    host        String,
    metric_name LowCardinality(String),
    value       Float64,
    collected_at DateTime64(3)
)
ENGINE = MergeTree()
ORDER BY (metric_name, collected_at);

INSERT INTO metrics
SELECT *
FROM file('metrics.arrow', Arrow);
```

## Writing to Arrow Format

Export a query result to an Arrow file:

```sql
SELECT metric_name, avg(value) AS avg_value, max(collected_at) AS latest
FROM metrics
GROUP BY metric_name
INTO OUTFILE 'metric_summary.arrow'
FORMAT Arrow;
```

From the shell:

```bash
clickhouse-client \
  --query "SELECT * FROM metrics FORMAT Arrow" \
  > metrics_export.arrow
```

## Using ArrowStream for Streaming Pipelines

The stream variant does not include a random-access footer, making it better for pipes and network sockets:

```bash
clickhouse-client \
  --query "SELECT * FROM metrics FORMAT ArrowStream" \
  | python3 consume_arrow_stream.py
```

On the Python side:

```python
import pyarrow as pa
import pyarrow.ipc as ipc
import sys

reader = ipc.open_stream(sys.stdin.buffer)
for batch in reader:
    df = batch.to_pandas()
    print(df.head())
```

## Reading Arrow Files Written by Python

Generate an Arrow file with PyArrow:

```python
import pyarrow as pa
import pyarrow.ipc as ipc

schema = pa.schema([
    ('id', pa.int64()),
    ('name', pa.string()),
    ('score', pa.float64()),
])

batch = pa.record_batch(
    [
        pa.array([1, 2, 3]),
        pa.array(['alice', 'bob', 'carol']),
        pa.array([9.5, 8.1, 7.7]),
    ],
    schema=schema,
)

with ipc.new_file('players.arrow', schema) as writer:
    writer.write_batch(batch)
```

Then read it in ClickHouse:

```sql
SELECT * FROM file('players.arrow', Arrow);
```

## Type Mapping

| ClickHouse Type | Arrow Type |
|-----------------|------------|
| Int8            | int8 |
| Int16           | int16 |
| Int32           | int32 |
| Int64           | int64 |
| UInt8           | uint8 |
| UInt32          | uint32 |
| UInt64          | uint64 |
| Float32         | float32 |
| Float64         | float64 |
| String          | utf8 |
| Date            | date32 |
| DateTime        | timestamp[s] |
| DateTime64(3)   | timestamp[ms] |
| Array(T)        | list |
| Nullable(T)     | nullable wrapper |

## Reading Arrow from S3

```sql
SELECT host, avg(value) AS avg_val
FROM s3(
    'https://my-bucket.s3.amazonaws.com/metrics/*.arrow',
    'ACCESS_KEY',
    'SECRET_KEY',
    'Arrow'
)
GROUP BY host;
```

## Performance Considerations

- Arrow has minimal serialization overhead because its in-memory layout is the wire format.
- For bulk transfers between services on the same machine or local network, Arrow and ArrowStream typically outperform Parquet due to no compression/decompression cycle.
- For long-term storage and cloud lake scenarios, Parquet or ORC are better choices due to their superior compression.
- When building a Python analytics pipeline that feeds ClickHouse, sending data as Arrow batches over HTTP is faster than converting to CSV or JSON first.

## Conclusion

Arrow and ArrowStream are the go-to formats when speed of data exchange between systems matters more than storage efficiency. If you are building pipelines in Python, R, or any language with an Arrow library, using Arrow format with ClickHouse eliminates the need for intermediate conversion steps.

**Related Reading:**

- [How to Use Parquet Format in ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-parquet-format/view)
- [How to Use Avro Format in ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-avro-format/view)
- [How to Export ClickHouse Data to Different File Formats](https://oneuptime.com/blog/post/2026-03-31-clickhouse-export-file-formats/view)
