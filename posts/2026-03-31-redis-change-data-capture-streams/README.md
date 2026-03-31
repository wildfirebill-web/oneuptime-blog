# How to Build a Change Data Capture System with Redis Streams

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Stream, Change Data Capture, CDC, Event Sourcing

Description: Learn how to build a Change Data Capture system using Redis Streams to propagate database changes to downstream consumers in real time.

---

Change Data Capture (CDC) captures every insert, update, and delete from a database and streams them to consumers. Redis Streams make an excellent CDC transport layer for fan-out to multiple downstream systems.

## CDC Architecture Overview

The pattern involves:
1. A CDC source (database triggers, Debezium, or application code) that writes changes to a Redis Stream
2. Consumer groups for each downstream system (search index, cache, analytics)
3. Consumers that process changes and update their respective systems

```text
PostgreSQL --> CDC Connector --> Redis Stream --> [Search Index Consumer]
                                              --> [Cache Invalidation Consumer]
                                              --> [Analytics Consumer]
```

## Writing Change Events

Whether using triggers, application code, or Debezium, change events follow a consistent schema:

```python
import redis
import json
import time

r = redis.Redis(decode_responses=True)

def emit_change_event(table, operation, row_id, before=None, after=None):
    """Emit a CDC event to the changes stream"""
    event = {
        'table': table,
        'operation': operation,   # INSERT, UPDATE, DELETE
        'row_id': str(row_id),
        'timestamp': int(time.time() * 1000),
        'before': json.dumps(before) if before else '',
        'after': json.dumps(after) if after else ''
    }
    stream_key = f'cdc:{table}'
    entry_id = r.xadd(stream_key, event, maxlen=500000, approximate=True)
    return entry_id

# Emit changes from application code
def update_product(product_id, new_price):
    old_product = db.get_product(product_id)
    db.update_product(product_id, price=new_price)
    new_product = db.get_product(product_id)

    emit_change_event(
        table='products',
        operation='UPDATE',
        row_id=product_id,
        before={'price': old_product.price},
        after={'price': new_price}
    )
```

## Setting Up Consumer Groups

Create a consumer group for each downstream system:

```bash
# Create consumer groups for different downstream systems
XGROUP CREATE cdc:products search-indexer $ MKSTREAM
XGROUP CREATE cdc:products cache-invalidator $ MKSTREAM
XGROUP CREATE cdc:products analytics $ MKSTREAM
```

## Cache Invalidation Consumer

```python
def cache_invalidation_consumer():
    group = 'cache-invalidator'
    stream = 'cdc:products'
    consumer = 'cache-worker-1'

    while True:
        msgs = r.xreadgroup(group, consumer, {stream: '>'}, count=20, block=5000)
        if not msgs:
            continue

        for _, entries in msgs:
            for msg_id, fields in entries:
                table = fields['table']
                row_id = fields['row_id']
                operation = fields['operation']

                cache_key = f'{table}:{row_id}'

                if operation in ('UPDATE', 'DELETE'):
                    r.delete(cache_key)
                    print(f"Invalidated cache for {cache_key}")

                r.xack(stream, group, msg_id)
```

## Search Index Consumer

```python
import elasticsearch

es = elasticsearch.Elasticsearch()

def search_index_consumer():
    group = 'search-indexer'
    stream = 'cdc:products'
    consumer = 'search-worker-1'

    while True:
        msgs = r.xreadgroup(group, consumer, {stream: '>'}, count=50, block=5000)
        if not msgs:
            continue

        for _, entries in msgs:
            for msg_id, fields in entries:
                operation = fields['operation']
                row_id = fields['row_id']

                if operation == 'DELETE':
                    es.delete(index='products', id=row_id, ignore=[404])
                else:
                    doc = json.loads(fields.get('after', '{}'))
                    es.index(index='products', id=row_id, body=doc)

                r.xack(stream, group, msg_id)
```

## Reading the Full Change History

Because streams persist entries, you can replay the full history for bootstrapping a new consumer:

```bash
# Read all changes from the beginning
XRANGE cdc:products - + COUNT 1000

# Bootstrap a new consumer group from the start
XGROUP CREATE cdc:products new-consumer 0 MKSTREAM
```

## Monitoring CDC Lag

```python
def get_cdc_lag(table):
    stream = f'cdc:{table}'
    groups = r.xinfo_groups(stream)
    return {g['name']: g.get('lag', 0) for g in groups}

lags = get_cdc_lag('products')
for group, lag in lags.items():
    print(f"{group}: {lag} unprocessed changes")
```

## Summary

Redis Streams work as a CDC transport by emitting change events to per-table streams, then using separate consumer groups for each downstream system (cache, search, analytics). Each consumer group processes changes independently and at its own pace. The stream's persistent log enables bootstrapping new consumers from history without re-reading the source database.
