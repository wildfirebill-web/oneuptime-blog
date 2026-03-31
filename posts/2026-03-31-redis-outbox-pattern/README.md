# How to Implement Outbox Pattern with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Outbox, Pattern, Stream, Reliability

Description: Implement the transactional outbox pattern using Redis Streams to guarantee that database writes and event publications are always in sync without distributed transactions.

---

When a service writes to a database and publishes an event, the two operations can get out of sync - the database write succeeds but the event is never published (or vice versa) if the process crashes in between. The outbox pattern solves this by writing both in a single local transaction and using a separate relay process to publish events reliably.

## The Problem

Naive approach - prone to split-brain:

```python
# WRONG: Two separate operations - one can succeed, the other can fail
db.execute("INSERT INTO orders ...")
r.xadd("events:orders", {"type": "order.created", ...})  # May never execute
```

## Outbox with Redis Streams

Write to the outbox atomically inside the database transaction, then relay to Redis:

```python
import psycopg2
import redis
import json

r = redis.Redis()

def create_order_with_outbox(conn, order_id, user_id, total):
    with conn.cursor() as cur:
        # Write the business record
        cur.execute("INSERT INTO orders (id, user_id, total) VALUES (%s, %s, %s)",
                    (order_id, user_id, total))
        # Write to the outbox table in the same transaction
        cur.execute("""INSERT INTO outbox (id, event_type, payload, created_at)
                       VALUES (%s, %s, %s, NOW())""",
                    (order_id, "order.created",
                     json.dumps({"order_id": order_id, "user_id": user_id, "total": total})))
    conn.commit()
```

## Outbox Relay Process

A separate relay process polls the outbox table and publishes to Redis:

```python
import time

def relay_outbox(conn):
    while True:
        with conn.cursor() as cur:
            cur.execute("""SELECT id, event_type, payload FROM outbox
                           WHERE published_at IS NULL
                           ORDER BY created_at LIMIT 100""")
            rows = cur.fetchall()
            for row_id, event_type, payload in rows:
                r.xadd(f"events:{event_type}", json.loads(payload),
                       maxlen=100000, approximate=True)
                cur.execute("UPDATE outbox SET published_at = NOW() WHERE id = %s",
                           (row_id,))
        conn.commit()
        if not rows:
            time.sleep(0.5)
```

## Idempotency at the Consumer

Since the relay may publish the same event twice after a crash, consumers must be idempotent:

```python
def process_event(msg_id, fields):
    event_id = fields.get(b"order_id", b"").decode()
    dedup_key = f"processed:order.created:{event_id}"
    # Set NX: only process if not already processed
    if not r.set(dedup_key, 1, ex=86400, nx=True):
        return  # Already processed
    # Process the event
    handle_order_created(fields)
```

## Redis-Only Outbox

For applications without a relational database, use a Redis list as the outbox:

```python
def write_with_redis_outbox(entity_key, entity_data, event_type, event_data):
    pipe = r.pipeline()
    pipe.set(entity_key, json.dumps(entity_data))
    pipe.rpush("outbox:pending", json.dumps({
        "event_type": event_type,
        "data": event_data,
        "ts": time.time()
    }))
    pipe.execute()
```

The relay drains the list and publishes to Streams:

```python
def relay_redis_outbox():
    while True:
        raw = r.lpop("outbox:pending")
        if not raw:
            time.sleep(0.5)
            continue
        event = json.loads(raw)
        r.xadd(f"events:{event['event_type']}", event["data"])
```

## Outbox Cleanup

Purge old published outbox rows to prevent unbounded growth:

```sql
DELETE FROM outbox WHERE published_at < NOW() - INTERVAL '7 days';
```

## Summary

The outbox pattern ensures database writes and event publications stay consistent by writing both in a single local transaction and relaying events asynchronously. Idempotent consumers handle the at-least-once delivery guarantee introduced by the relay process, giving you reliable event publishing without distributed transactions or two-phase commits.
