# How to Use ClickHouse with Singer Taps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Singer, Tap, Target, ELT, Data Integration

Description: Use Singer taps to extract data from any source and load it into ClickHouse using a Singer target, enabling open-source ELT pipelines.

---

Singer is an open-source standard for writing scripts that move data between sources (taps) and destinations (targets). Building a Singer pipeline that loads into ClickHouse gives you a vendor-neutral ELT stack.

## Singer Concepts

- **Tap**: reads data from a source and outputs JSON records to stdout
- **Target**: reads those records from stdin and loads them into a destination
- **Catalog**: describes the schema of streams the tap can extract

## Installing a Tap

```bash
pip install tap-github
tap-github --config tap_config.json --discover > catalog.json
```

Example `tap_config.json`:

```text
{
  "access_token": "your_github_token",
  "repository": "your-org/your-repo",
  "start_date": "2024-01-01T00:00:00Z"
}
```

## Building a Custom Singer Target for ClickHouse

Create a Python target that reads Singer records and inserts them into ClickHouse.

```python
#!/usr/bin/env python3
import sys
import json
import clickhouse_connect
from collections import defaultdict

client = clickhouse_connect.get_client(
    host='localhost',
    port=8123,
    username='default',
    password=''
)

buffers = defaultdict(list)

for line in sys.stdin:
    msg = json.loads(line)
    if msg['type'] == 'RECORD':
        stream = msg['stream']
        buffers[stream].append(msg['record'])
        if len(buffers[stream]) >= 1000:
            flush_buffer(stream, buffers[stream])
            buffers[stream] = []
    elif msg['type'] == 'STATE':
        # Emit state for checkpointing
        print(json.dumps(msg), flush=True)

# Flush remaining records
for stream, records in buffers.items():
    if records:
        flush_buffer(stream, records)
```

```python
def flush_buffer(stream: str, records: list):
    if not records:
        return
    keys = list(records[0].keys())
    rows = [[r.get(k) for k in keys] for r in records]
    client.insert(
        f"default.{stream}",
        rows,
        column_names=keys
    )
    print(f"Inserted {len(records)} rows into {stream}", file=sys.stderr)
```

## Running the Pipeline

Connect a tap to the ClickHouse target using a Unix pipe:

```bash
tap-github --config tap_config.json --catalog catalog.json \
  | target-clickhouse --config target_config.json
```

## Auto-Creating Tables

Before inserting, generate a CREATE TABLE statement from Singer schema:

```python
def create_table_if_not_exists(stream: str, schema: dict):
    props = schema.get('properties', {})
    columns = []
    for col, info in props.items():
        types = info.get('type', ['string'])
        if 'integer' in types:
            ch_type = 'Nullable(Int64)'
        elif 'number' in types:
            ch_type = 'Nullable(Float64)'
        else:
            ch_type = 'Nullable(String)'
        columns.append(f"`{col}` {ch_type}")
    ddl = f"""
        CREATE TABLE IF NOT EXISTS default.{stream}
        ({', '.join(columns)})
        ENGINE = MergeTree()
        ORDER BY tuple()
    """
    client.command(ddl)
```

## Summary

Singer's tap-and-target model makes ClickHouse a plug-and-play destination for hundreds of data sources. By writing a Singer target for ClickHouse, any Singer-compatible tap can load data into your analytical database without custom integration work per source.
