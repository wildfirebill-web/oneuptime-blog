# How to Automate ClickHouse Table Creation from JSON Schema

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Automation, JSON Schema, Schema Migration, Python

Description: A guide to automating ClickHouse table creation by converting JSON Schema definitions into CREATE TABLE statements using Python, with type mapping and MergeTree configuration.

---

## Why Automate Table Creation

When your application schema is defined in JSON Schema (for API validation or data contracts), manually translating it into ClickHouse DDL is error-prone and tedious. Automating this translation ensures consistency and makes schema evolution trackable.

## JSON Schema to ClickHouse Type Mapping

```text
JSON Schema type    | ClickHouse type
--------------------|------------------
integer             | Int64
number              | Float64
string              | String
boolean             | UInt8
string (date-time)  | DateTime64(3)
string (date)       | Date
array               | Array(String)
null + type         | Nullable(type)
```

## Python Script: JSON Schema to ClickHouse DDL

```python
import json

TYPE_MAP = {
    "integer": "Int64",
    "number": "Float64",
    "string": "String",
    "boolean": "UInt8",
}

FORMAT_MAP = {
    "date-time": "DateTime64(3)",
    "date": "Date",
}

def json_schema_to_clickhouse(schema: dict, table_name: str,
                               order_by: list[str],
                               partition_by: str = None) -> str:
    properties = schema.get("properties", {})
    required = set(schema.get("required", []))
    columns = []

    for col_name, col_def in properties.items():
        col_type = col_def.get("type", "string")
        col_format = col_def.get("format", "")

        if col_format in FORMAT_MAP:
            ch_type = FORMAT_MAP[col_format]
        elif isinstance(col_type, list) and "null" in col_type:
            base = [t for t in col_type if t != "null"][0]
            ch_type = f"Nullable({TYPE_MAP.get(base, 'String')})"
        else:
            ch_type = TYPE_MAP.get(col_type, "String")

        columns.append(f"  {col_name}  {ch_type}")

    ddl = f"CREATE TABLE IF NOT EXISTS {table_name} (\n"
    ddl += ",\n".join(columns)
    ddl += "\n) ENGINE = MergeTree()\n"

    if partition_by:
        ddl += f"PARTITION BY {partition_by}\n"

    ddl += f"ORDER BY ({', '.join(order_by)});"
    return ddl
```

## Example Usage

```python
schema = {
    "type": "object",
    "required": ["event_id", "user_id", "event_time"],
    "properties": {
        "event_id":   {"type": "integer"},
        "user_id":    {"type": "integer"},
        "event_type": {"type": "string"},
        "event_time": {"type": "string", "format": "date-time"},
        "amount":     {"type": ["number", "null"]},
        "is_mobile":  {"type": "boolean"}
    }
}

ddl = json_schema_to_clickhouse(
    schema,
    table_name="analytics.events",
    order_by=["user_id", "event_time"],
    partition_by="toYYYYMM(event_time)"
)
print(ddl)
```

Output:

```sql
CREATE TABLE IF NOT EXISTS analytics.events (
  event_id    Int64,
  user_id     Int64,
  event_type  String,
  event_time  DateTime64(3),
  amount      Nullable(Float64),
  is_mobile   UInt8
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_time)
ORDER BY (user_id, event_time);
```

## Executing the DDL

```python
import clickhouse_connect

client = clickhouse_connect.get_client(host='localhost')
client.command(ddl)
print(f"Table {table_name} created successfully.")
```

## CI/CD Integration

```bash
# In your CI pipeline, validate schema and generate DDL
python generate_ddl.py --schema schemas/events.json \
  --table analytics.events \
  --order-by user_id event_time \
  --partition-by "toYYYYMM(event_time)" \
  | clickhouse-client --multiquery
```

## Summary

Automating ClickHouse table creation from JSON Schema involves mapping JSON types to ClickHouse types, handling nullable types, and appending MergeTree-specific clauses like `ORDER BY` and `PARTITION BY`. A simple Python script can translate a JSON Schema definition into valid DDL, enabling schema-as-code workflows where table structure is versioned alongside your API contracts.
