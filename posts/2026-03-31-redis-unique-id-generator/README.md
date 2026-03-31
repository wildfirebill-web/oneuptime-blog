# How to Implement a Unique ID Generator with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, ID, Generator

Description: Build a distributed unique ID generator with Redis using atomic counters and Snowflake-style IDs for high-throughput, collision-free identifier generation.

---

Generating unique IDs across a distributed system is trickier than it looks. UUIDs are fine but large and random. Database auto-increment requires a single write path. Redis atomic counters give you sortable, compact IDs that work across multiple application servers.

## Simple Sequential ID Generator

Use INCR for per-entity sequential IDs:

```python
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def next_id(entity_type: str) -> int:
    """Returns a monotonically increasing integer ID for the given entity type."""
    return r.incr(f"id_seq:{entity_type}")

# Usage
order_id = next_id("order")       # 1, 2, 3, ...
user_id = next_id("user")         # 1, 2, 3, ...
invoice_id = next_id("invoice")   # independent sequence
```

## Batch ID Allocation

For bulk inserts, reserve a range of IDs at once:

```python
def reserve_id_range(entity_type: str, count: int) -> range:
    """Returns a range of `count` IDs reserved for exclusive use."""
    end = r.incrby(f"id_seq:{entity_type}", count)
    start = end - count + 1
    return range(start, end + 1)

# Reserve 100 IDs for bulk order import
ids = reserve_id_range("order", 100)
for order_id in ids:
    process_order(order_id)
```

## Snowflake-Style Composite IDs

For distributed systems, combine a timestamp, worker ID, and a per-worker sequence counter to generate globally unique, time-sortable IDs:

```python
import time

EPOCH = 1700000000  # custom epoch (seconds)
WORKER_BITS = 10
SEQUENCE_BITS = 12
MAX_SEQUENCE = (1 << SEQUENCE_BITS) - 1  # 4095

def generate_snowflake_id(worker_id: int) -> int:
    ts = int(time.time() * 1000) - EPOCH * 1000  # milliseconds since epoch

    # Atomic sequence increment, reset every millisecond
    seq_key = f"snowflake:{worker_id}:{ts}"
    sequence = r.incr(seq_key)
    r.expire(seq_key, 1)  # auto-cleanup after 1 second

    if sequence > MAX_SEQUENCE:
        # Wait for next millisecond
        time.sleep(0.001)
        return generate_snowflake_id(worker_id)

    # Compose: [timestamp 41 bits][worker 10 bits][sequence 12 bits]
    snowflake = (ts << (WORKER_BITS + SEQUENCE_BITS)) | (worker_id << SEQUENCE_BITS) | sequence
    return snowflake

def decode_snowflake(snowflake_id: int) -> dict:
    sequence = snowflake_id & MAX_SEQUENCE
    worker_id = (snowflake_id >> SEQUENCE_BITS) & ((1 << WORKER_BITS) - 1)
    ts = (snowflake_id >> (WORKER_BITS + SEQUENCE_BITS)) + EPOCH * 1000
    return {
        "timestamp_ms": ts,
        "worker_id": worker_id,
        "sequence": sequence
    }
```

## Prefixed Human-Readable IDs

Combine sequential IDs with a prefix for readable identifiers:

```python
def generate_prefixed_id(prefix: str) -> str:
    """Generates IDs like ORD-00001234, INV-00005678."""
    sequence = r.incr(f"id_seq:{prefix.lower()}")
    return f"{prefix.upper()}-{sequence:08d}"

# Examples
order_ref = generate_prefixed_id("ORD")    # ORD-00000001
invoice_ref = generate_prefixed_id("INV")  # INV-00000001
```

## Persisting Sequences Across Restarts

Since Redis is in-memory, initialize sequence counters from your database on startup:

```python
def init_sequences_from_db(db_connection):
    cursor = db_connection.cursor()
    cursor.execute("SELECT entity_type, MAX(id) FROM all_ids GROUP BY entity_type")
    pipe = r.pipeline()
    for entity_type, max_id in cursor.fetchall():
        # Only set if Redis value is lower (Redis may have been updated more recently)
        current = r.get(f"id_seq:{entity_type}")
        if current is None or int(current) < max_id:
            pipe.set(f"id_seq:{entity_type}", max_id)
    pipe.execute()
```

## Summary

Redis INCR provides the simplest distributed sequential ID generator available. For high-throughput distributed environments, the Snowflake-style pattern adds time-sortability without coordination overhead. Always initialize counters from your database on service restart to avoid ID collisions after a Redis failover.
