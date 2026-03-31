# How to Use ClickHouse Python Client (clickhouse-connect)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Python, clickhouse-connect, Data Ingestion, Query

Description: Learn how to install and use the clickhouse-connect Python client to query ClickHouse, insert data, and stream results with Pandas integration.

---

`clickhouse-connect` is the officially recommended Python client for ClickHouse. It communicates over HTTP and offers tight Pandas integration, streaming queries, and support for all ClickHouse data types.

## Installation

```bash
pip install clickhouse-connect
```

## Connecting to ClickHouse

```python
import clickhouse_connect

client = clickhouse_connect.get_client(
    host='localhost',
    port=8123,
    username='default',
    password='',
    database='default'
)
```

For ClickHouse Cloud, use the secure flag:

```python
client = clickhouse_connect.get_client(
    host='abc123.clickhouse.cloud',
    port=8443,
    username='default',
    password='your_password',
    secure=True
)
```

## Running Queries

Use `query()` to retrieve results as a `QueryResult` object:

```python
result = client.query('SELECT number, number * number AS square FROM numbers(5)')
print(result.column_names)
print(result.result_rows)
```

For row-by-row iteration on large datasets, use `query_row_block_stream()`:

```python
with client.query_row_block_stream('SELECT * FROM large_table') as stream:
    for block in stream:
        for row in block:
            print(row)
```

## Pandas Integration

Fetch results directly into a DataFrame:

```python
df = client.query_df('SELECT event_date, count() AS cnt FROM events GROUP BY event_date')
print(df.head())
```

Insert a DataFrame:

```python
import pandas as pd

df = pd.DataFrame({
    'id': [1, 2, 3],
    'name': ['alice', 'bob', 'carol'],
    'score': [95.5, 87.2, 91.0]
})
client.insert_df('users', df)
```

## Inserting Data

Use `insert()` with a list of rows:

```python
data = [
    [1, '2024-01-01', 'pageview'],
    [2, '2024-01-02', 'click'],
    [3, '2024-01-03', 'purchase'],
]
client.insert('events', data, column_names=['id', 'event_date', 'event_type'])
```

## Command Execution (DDL)

Use `command()` for DDL statements that return no rows:

```python
client.command('''
    CREATE TABLE IF NOT EXISTS events (
        id UInt64,
        event_date Date,
        event_type String
    ) ENGINE = MergeTree()
    ORDER BY (event_date, id)
''')
```

## Query with Parameters

Use named parameters to safely pass values:

```python
result = client.query(
    'SELECT * FROM events WHERE event_type = {et:String} AND event_date >= {d:Date}',
    parameters={'et': 'pageview', 'd': '2024-01-01'}
)
```

## Connection Settings

```python
client = clickhouse_connect.get_client(
    host='localhost',
    connect_timeout=10,
    send_receive_timeout=300,
    compress=True,          # enable HTTP compression
    query_limit=0           # no row limit
)
```

## Summary

`clickhouse-connect` is the recommended Python client for ClickHouse, offering HTTP-based connectivity, first-class Pandas support, streaming queries, and safe parameterized queries. It suits both analytical workflows and production data pipelines that use Python.
