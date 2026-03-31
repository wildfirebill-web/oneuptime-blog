# How to Listen for Key Expiration Events in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Keyspace Notification, Key Expiration, TTL, Event

Description: Subscribe to Redis key expiration events using keyspace notifications to trigger application logic when keys with TTLs expire.

---

## Use Cases for Key Expiration Events

Listening for key expiration allows you to react when a key's TTL reaches zero:

- Session cleanup: remove session-related data when a session key expires
- Scheduler: use a key's TTL as a timer, trigger logic on expiry
- Cache invalidation: notify dependent systems when a cached item expires
- Lease management: release distributed locks or leases on expiry
- Audit logging: record when ephemeral data is removed

## Step 1 - Enable Expiry Event Notifications

```bash
# Enable keyevent notifications for expired events only
redis-cli CONFIG SET notify-keyspace-events "Ex"

# Or enable keyspace events for expired events too
redis-cli CONFIG SET notify-keyspace-events "KEx"
```

Flags used:
- `E` - Keyevent notifications (`__keyevent@<db>__:<event>`)
- `x` - Expired events (TTL reached zero)

Persist in `redis.conf`:

```text
notify-keyspace-events Ex
```

## Step 2 - Subscribe to the Expiry Channel

```bash
redis-cli SUBSCRIBE __keyevent@0__:expired
```

The channel `__keyevent@0__:expired` delivers the name of every key that expires in database 0.

Test it:

```bash
# Terminal 1: subscriber
redis-cli SUBSCRIBE __keyevent@0__:expired

# Terminal 2: create a short-lived key
redis-cli SET tempkey "hello" EX 5
# After 5 seconds, terminal 1 receives:
# 1) "message"
# 2) "__keyevent@0__:expired"
# 3) "tempkey"
```

## Step 3 - Listen for Expiry in Python

```python
import redis
import threading

r = redis.Redis(host='localhost', port=6379)

# Enable expiry events
r.config_set('notify-keyspace-events', 'Ex')

def on_key_expired(key_name):
    print(f"Key expired: {key_name}")
    # Add your logic here:
    # - Remove related data
    # - Send notifications
    # - Trigger workflows

def listen_for_expirations():
    sub = redis.Redis(host='localhost', port=6379)
    pubsub = sub.pubsub()
    pubsub.subscribe('__keyevent@0__:expired')

    for message in pubsub.listen():
        if message['type'] == 'message':
            key = message['data']
            if isinstance(key, bytes):
                key = key.decode()
            on_key_expired(key)

thread = threading.Thread(target=listen_for_expirations, daemon=True)
thread.start()

# Create an expiring key
r.set('session:user123', 'active', ex=10)
print("Waiting for expiry...")

import time; time.sleep(15)
```

## Step 4 - Handle Multiple Databases

If you use multiple Redis databases, subscribe to each:

```python
# Subscribe to expired events in databases 0 and 1
pubsub.subscribe(
    '__keyevent@0__:expired',
    '__keyevent@1__:expired',
)

# Or use a pattern to subscribe to all databases
pubsub.psubscribe('__keyevent@*__:expired')

for message in pubsub.listen():
    if message['type'] == 'pmessage':
        channel = message['channel'].decode()
        db = channel.split('@')[1].split('_')[0]  # Extract database number
        key = message['data'].decode()
        print(f"DB {db}: Key expired: {key}")
```

## Step 5 - Important Caveat: Delayed or Missed Expirations

Redis expires keys lazily (on access) and with a background sweep. A key may not trigger an expiry event exactly at its TTL - there can be a delay of up to a few seconds depending on the background sweep frequency.

Additionally:

- If Redis restarts, expiry events for keys that expired during downtime are not replayed
- Expiry events are not guaranteed delivery (Pub/Sub is fire-and-forget)
- Replicas do not independently generate expiry events; the primary sends the event and the replica replicates the DEL

```bash
# Check active expire cycle settings
redis-cli CONFIG GET hz
# Default: 10 (background expires 10x/sec)

# Increase for more frequent expiry sweeps (more CPU usage)
redis-cli CONFIG SET hz 20
```

## Step 6 - Pattern-Based Key Filtering

If you only care about specific key patterns expiring, filter in your subscriber:

```python
def on_key_expired(key_name):
    # Only handle session keys
    if key_name.startswith('session:'):
        user_id = key_name.split(':')[1]
        cleanup_user_session(user_id)
    elif key_name.startswith('lock:'):
        lock_name = key_name.split(':', 1)[1]
        release_lock(lock_name)
```

## Step 7 - Using Expiry Events as a Scheduler

Use key TTL as a countdown timer for delayed tasks:

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379)
r.config_set('notify-keyspace-events', 'Ex')

def schedule_task(task_id, task_data, delay_seconds):
    key = f'scheduled:{task_id}'
    r.set(key, json.dumps(task_data), ex=delay_seconds)
    print(f"Task {task_id} scheduled in {delay_seconds}s")

def on_key_expired(key_name):
    if key_name.startswith('scheduled:'):
        task_id = key_name.split(':', 1)[1]
        print(f"Executing scheduled task: {task_id}")
        # Note: the value is gone when we receive this event
        # Store task data in a separate key or external store

schedule_task('report-001', {'type': 'report', 'user': 123}, delay_seconds=30)
```

**Note:** When an expiry event fires, the key has already been deleted. You cannot read the expired key's value in the event handler. Store necessary data in a companion key with a longer TTL or in an external store.

## Summary

Listen for Redis key expiration events by enabling `notify-keyspace-events Ex` and subscribing to `__keyevent@0__:expired`. This enables TTL-based workflows like session cleanup, delayed task execution, and distributed lease management. Be aware that expiry notifications may be delayed by several seconds due to the lazy expiry mechanism, are not persisted, and the key's value is unavailable when the event fires - so store any needed metadata separately before the key expires.
