# How to Troubleshoot Redis Client Buffer Overflow

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Client Buffer, Memory, Troubleshooting, Performance

Description: Diagnose and fix Redis client output buffer overflow errors that cause client disconnections, by tuning buffer limits and identifying slow consumers.

---

## What is Client Buffer Overflow

Redis maintains an output buffer for each connected client to hold responses being sent. If the client reads responses too slowly, or if Redis sends a very large response (e.g., KEYS *, large LRANGE), the buffer grows. When it exceeds the configured limit, Redis forcibly disconnects the client.

You will see this in logs:

```text
Client id=123 addr=192.168.1.5:54321 fd=8 name= age=120 idle=60 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=100000 omem=167772160 events=rw cmd=subscribe disconnected
```

`omem` is the output memory used by that client. When it exceeds the hard limit, the client is disconnected.

## Step 1 - Check Current Buffer Limits

```bash
redis-cli CONFIG GET client-output-buffer-limit
```

Default output:

```text
client-output-buffer-limit
normal 0 0 0
slave 268435456 67108864 60
pubsub 33554432 8388608 60
```

Format: `class hard_limit soft_limit soft_limit_seconds`

- `normal`: Regular clients. Default `0 0 0` means unlimited.
- `slave`: Replica connections. Hard limit 256MB, soft limit 64MB for 60 seconds.
- `pubsub`: Pub/Sub subscribers. Hard limit 32MB, soft limit 8MB for 60 seconds.

## Step 2 - Identify Clients with Large Buffers

```bash
redis-cli CLIENT LIST | grep -v "omem=0" | sort -t= -k15 -rn | head -20
```

Or use INFO clients:

```bash
redis-cli INFO clients | grep -E 'blocked_clients|client_recent_max_output_buffer'
```

## Step 3 - Identify Slow Consumers

The root cause of buffer overflow is usually a slow consumer. Check:

1. Pub/Sub subscribers that are not reading messages fast enough
2. Clients running MONITOR (always high output)
3. Clients receiving large responses that process them slowly

```bash
# Find MONITOR clients (very high output volume)
redis-cli CLIENT LIST | grep "cmd=monitor"
```

Kill a misbehaving client:

```bash
redis-cli CLIENT KILL ID 123
```

## Step 4 - Tune Buffer Limits

For Pub/Sub subscribers that legitimately need more buffer space:

```bash
redis-cli CONFIG SET client-output-buffer-limit "pubsub 64mb 16mb 120"
```

For normal clients if you have an application that requests large data chunks:

```bash
redis-cli CONFIG SET client-output-buffer-limit "normal 256mb 64mb 60"
```

Persist in `redis.conf`:

```text
client-output-buffer-limit normal 256mb 64mb 60
client-output-buffer-limit slave 512mb 128mb 120
client-output-buffer-limit pubsub 64mb 16mb 120
```

## Step 5 - Avoid Commands that Return Large Responses

Replace `KEYS *` with `SCAN`:

```bash
# Bad - blocks and returns all keys at once
KEYS *

# Good - iterates in batches
SCAN 0 COUNT 100
```

Replace `LRANGE key 0 -1` (full list) with paginated reads:

```bash
# Read in chunks of 1000
LRANGE mylist 0 999
LRANGE mylist 1000 1999
```

In Python:

```python
import redis

r = redis.Redis(host='localhost', port=6379)

# Paginated SCAN instead of KEYS
cursor = 0
while True:
    cursor, keys = r.scan(cursor=cursor, count=100)
    for key in keys:
        process(key)
    if cursor == 0:
        break
```

## Step 6 - Use Pipelining to Batch Reads

When sending many commands, pipeline them to reduce round-trips. This also reduces the chance that slow network delivery causes buffer growth:

```python
pipe = r.pipeline(transaction=False)
for i in range(1000):
    pipe.get(f'key:{i}')
results = pipe.execute()
```

## Step 7 - Monitor Output Buffer Usage

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379)

while True:
    clients = r.client_list()
    high_buffer = [c for c in clients if int(c.get('omem', 0)) > 1_000_000]
    for c in high_buffer:
        print(f"Client {c['id']} addr={c['addr']} cmd={c['cmd']} omem={c['omem']}")
    time.sleep(5)
```

Alert when any client has `omem` above a threshold (e.g., 10MB for Pub/Sub, 50MB for normal clients).

## Summary

Redis client buffer overflow is caused by consumers reading responses slower than Redis sends them. Identify slow consumers with `CLIENT LIST`, kill MONITOR clients or slow Pub/Sub subscribers, and tune `client-output-buffer-limit` for each client class. Avoid large single-response commands like `KEYS *` and `LRANGE key 0 -1` in favour of `SCAN` and paginated reads to prevent buffer growth in the first place.
