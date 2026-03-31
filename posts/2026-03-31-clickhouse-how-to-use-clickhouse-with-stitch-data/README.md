# How to Use ClickHouse with Stitch Data

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Stitch, ELT, Data Integration, Replication, ETL

Description: Configure Stitch Data to replicate data from SaaS applications and databases into ClickHouse, enabling fast analytical queries on replicated datasets.

---

Stitch (now part of Talend) is a cloud-based ELT service with pre-built connectors for databases, SaaS tools, and event streams. Adding ClickHouse as a destination lets you consolidate data from many sources for fast analytics.

## Setting Up ClickHouse as a Stitch Destination

Stitch supports ClickHouse via its generic HTTP or JDBC destination path. Since native ClickHouse support may vary by Stitch plan, the most reliable approach uses ClickHouse's PostgreSQL compatibility layer or a custom destination via Stitch's Import API.

### Option 1 - Using the Stitch Import API

Stitch provides a REST API for pushing data to any destination. Use it with a ClickHouse consumer.

```python
import requests
import json
import clickhouse_connect
import pandas as pd

STITCH_API_TOKEN = "your_stitch_api_token"
CLIENT_ID = 123456

def push_to_stitch(records: list, table_name: str):
    payload = {
        "table_name": table_name,
        "schema": {
            "properties": {
                "id": {"type": ["integer"]},
                "name": {"type": ["null", "string"]},
                "created_at": {"type": ["null", "string"], "format": "date-time"}
            }
        },
        "messages": [
            {"action": "upsert", "data": r, "sequence": i}
            for i, r in enumerate(records)
        ],
        "key_names": ["id"]
    }
    response = requests.post(
        f"https://api.stitchdata.com/v2/import/batch",
        headers={
            "Authorization": f"Bearer {STITCH_API_TOKEN}",
            "Content-Type": "application/json"
        },
        data=json.dumps(payload)
    )
    response.raise_for_status()
    return response.json()
```

### Option 2 - Reading from Stitch S3 Output

Configure Stitch to write to S3, then load from S3 into ClickHouse:

```sql
INSERT INTO default.users
SELECT *
FROM s3(
    'https://s3.amazonaws.com/your-stitch-bucket/users/*.jsonl',
    'AWS_KEY',
    'AWS_SECRET',
    'JSONEachRow'
);
```

## Loading Data Incrementally

For incremental loads, track a watermark and only load new records:

```sql
INSERT INTO default.users
SELECT *
FROM s3(
    'https://s3.amazonaws.com/your-stitch-bucket/users/*.jsonl',
    'AWS_KEY', 'AWS_SECRET', 'JSONEachRow'
)
WHERE updated_at > (SELECT max(updated_at) FROM default.users);
```

## Deduplicating Stitch Records in ClickHouse

Stitch may send duplicate records for updated rows. Use ReplacingMergeTree to handle this:

```sql
CREATE TABLE default.users (
    id UInt64,
    name String,
    email String,
    updated_at DateTime
) ENGINE = ReplacingMergeTree(updated_at)
ORDER BY id;
```

Query with deduplication:

```sql
SELECT id, name, email, updated_at
FROM default.users FINAL
WHERE email != '';
```

## Summary

Stitch Data simplifies ELT from many sources. When you route Stitch output to ClickHouse, either via the Import API, S3 staging, or a compatibility layer, you can build fast analytics on consolidated data. Use ReplacingMergeTree to cleanly handle the upsert patterns that Stitch produces.
