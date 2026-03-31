# How to Use BSONEachRow Format in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, BSON, MongoDB, Data Engineering

Description: Learn how to use ClickHouse's BSONEachRow format to migrate data from MongoDB, read BSON exports, and map BSON types to ClickHouse column types.

## What Is BSONEachRow?

BSON (Binary JSON) is the binary serialization format used by MongoDB. It extends JSON with additional data types like ObjectId, Date, Binary, Decimal128, and more. ClickHouse's `BSONEachRow` format reads and writes a stream of BSON documents - one document per row - making it the natural format for MongoDB data migration.

`BSONEachRow` is analogous to `JSONEachRow` but uses the BSON wire format instead of text JSON.

## When to Use BSONEachRow

- **MongoDB migration**: `mongoexport` can dump collections as BSON, which ClickHouse reads directly.
- **Application integration**: Any application using a MongoDB driver can produce BSON for ClickHouse.
- **Preserving types**: BSON preserves type information (dates, binary, integers) that JSON loses.

## Exporting from MongoDB

Use `mongodump` to create a BSON dump:

```bash
mongodump \
  --uri="mongodb://localhost:27017" \
  --db=myapp \
  --collection=orders \
  --out=/tmp/mongo_dump
```

This creates `/tmp/mongo_dump/myapp/orders.bson`.

Alternatively, use `mongoexport` with BSON output (note: `mongoexport` defaults to JSON; use `bsondump` to convert):

```bash
bsondump /tmp/mongo_dump/myapp/orders.bson > /tmp/orders_bson_stream.bson
```

## Reading BSON in ClickHouse

```sql
SELECT *
FROM file('/tmp/orders_bson_stream.bson', BSONEachRow)
LIMIT 10;
```

Inspect the inferred schema:

```sql
DESCRIBE file('/tmp/orders_bson_stream.bson', BSONEachRow);
```

## Creating a Table and Loading BSON Data

```sql
CREATE TABLE orders
(
    order_id    String,     -- MongoDB ObjectId stored as string
    customer_id String,
    status      LowCardinality(String),
    total       Float64,
    items       String,     -- Nested array stored as JSON string
    created_at  DateTime64(3)
)
ENGINE = MergeTree()
ORDER BY (created_at, customer_id);

INSERT INTO orders
SELECT *
FROM file('/tmp/orders_bson_stream.bson', BSONEachRow);
```

## BSON to ClickHouse Type Mapping

| BSON Type | ClickHouse Type |
|-----------|-----------------|
| Double | Float64 |
| String | String |
| Document (embedded) | String (JSON) or Map |
| Array | Array(T) or String |
| Binary | String |
| ObjectId | FixedString(12) or String |
| Boolean | UInt8 |
| Date (UTC datetime) | DateTime64(3) |
| Null | Nullable(T) |
| Int32 | Int32 |
| Int64 | Int64 |
| Decimal128 | Decimal(38, 10) |

## Handling ObjectId

MongoDB ObjectIds are 12-byte binary values. When reading BSON, ClickHouse represents them as hex strings. Map them to `String` or `FixedString(24)`:

```sql
CREATE TABLE mongo_users
(
    _id    FixedString(24), -- 24 hex characters = 12 bytes
    name   String,
    email  String,
    age    UInt8
)
ENGINE = MergeTree()
ORDER BY _id;
```

## Writing BSON from ClickHouse

Export ClickHouse data as BSON for consumption by MongoDB or a BSON-capable application:

```sql
SELECT order_id, customer_id, total, created_at
FROM orders
INTO OUTFILE 'orders_export.bson'
FORMAT BSONEachRow;
```

From the shell:

```bash
clickhouse-client \
  --query "SELECT * FROM orders FORMAT BSONEachRow" \
  > orders_export.bson
```

## Handling Nested Documents

BSON documents can contain nested documents (subdocuments). ClickHouse can map them as:

1. **String** - stores the nested document as a JSON string
2. **Map(String, String)** - flattens one level of nesting
3. **Tuple** - maps to a fixed-schema nested structure

Enable object reading as strings:

```sql
SET input_format_bson_skip_fields_with_unsupported_types_in_schema_inference = 1;
```

Then extract nested fields using JSON functions:

```sql
SELECT
    order_id,
    JSONExtractString(shipping_address, 'city') AS city,
    JSONExtractString(shipping_address, 'country') AS country
FROM orders;
```

## Practical Migration Workflow

A complete MongoDB to ClickHouse migration:

```bash
# Step 1: Dump the MongoDB collection
mongodump --uri="mongodb://localhost:27017" \
  --db=ecommerce --collection=orders \
  --out=/tmp/dump

# Step 2: Convert to a BSON stream
bsondump /tmp/dump/ecommerce/orders.bson \
  --outFile=/tmp/orders.bson

# Step 3: Load into ClickHouse
clickhouse-client \
  --query "INSERT INTO orders FORMAT BSONEachRow" \
  < /tmp/orders.bson
```

## Performance Tips

1. BSON is a binary format, so it parses faster than JSON for the same data.
2. For large MongoDB collections, split the BSON dump into chunks and load in parallel.
3. Use `mongodump` with `--numParallelCollections` to speed up the initial export.
4. After migration, convert your ClickHouse table to use proper types (not `String` for everything) for better query performance.

## Conclusion

`BSONEachRow` makes migrating from MongoDB to ClickHouse straightforward. You can take a standard MongoDB dump and load it directly without a conversion step. Once the data is in ClickHouse, you gain full SQL analytical capabilities with columnar storage performance.

**Related Reading:**

- [How to Use JSONEachRow Format in ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-jsoneachrow-format/view)
- [How to Use Protobuf Format in ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-protobuf-format/view)
- [How to Import Data from S3 in Various Formats in ClickHouse](https://oneuptime.com/blog/post/2026-03-31-clickhouse-import-from-s3/view)
