# How to Troubleshoot Redis Pub/Sub Message Loss

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Pub/Sub, Messaging, Troubleshooting, Queue

Description: Diagnose and fix Redis Pub/Sub message loss caused by slow subscribers, client buffer overflow, and network disconnections.

---

Redis Pub/Sub delivers messages to subscribers in real time, but it has no persistence. If a subscriber is too slow, disconnected, or the output buffer overflows, messages are silently dropped. This guide covers how to detect and fix these issues.

## Understand Why Redis Pub/Sub Drops Messages

Unlike Redis Streams, Pub/Sub is fire-and-forget:
- No message backlog or replay
- Slow subscribers get disconnected when their output buffer is full
- Disconnected clients miss all messages published while offline

## Detect Dropped Messages with PUBSUB Stats

```bash
# List active channels
redis-cli PUBSUB CHANNELS "*"

# Count subscribers per channel
redis-cli PUBSUB NUMSUB my-channel

# Count pattern subscribers
redis-cli PUBSUB NUMPAT
```

If subscriber count drops unexpectedly, clients are disconnecting.

## Check Client Output Buffer Limits

When a subscriber processes messages slower than they arrive, its output buffer fills up. Redis disconnects clients that exceed the configured limit:

```bash
redis-cli INFO clients | grep client_recent_max_output_buffer
redis-cli CLIENT LIST | grep "cmd=subscribe"
```

Check the configured limits:

```bash
redis-cli CONFIG GET client-output-buffer-limit
```

Default output for Pub/Sub:

```text
pubsub 33554432 8388608 60
```

This means: hard limit 32MB, soft limit 8MB sustained for 60 seconds. Increase limits for slow subscribers:

```bash
redis-cli CONFIG SET client-output-buffer-limit "pubsub 67108864 16777216 120"
```

## Monitor Client Disconnections

Check Redis logs for disconnected pub/sub clients:

```bash
journalctl -u redis | grep "Client disconnected\|close"
```

Enable verbose logging temporarily:

```bash
redis-cli CONFIG SET loglevel verbose
```

## Switch to Redis Streams for Reliable Messaging

If message loss is unacceptable, migrate from Pub/Sub to Streams:

```bash
# Producer
redis-cli XADD events "*" action "user.login" user_id "42"

# Consumer with persistence
redis-cli XREAD COUNT 10 STREAMS events 0-0
```

Streams provide:
- Message persistence
- Consumer groups with acknowledgments
- Ability to replay messages after reconnection

## Fix Subscriber Performance

The root cause is often a slow subscriber. Optimize by processing messages asynchronously:

```python
import redis
import threading

r = redis.Redis()
pubsub = r.pubsub()
pubsub.subscribe("my-channel")

def process_message(message):
    # Do heavy processing in a thread pool, not inline
    pass

for message in pubsub.listen():
    if message["type"] == "message":
        threading.Thread(target=process_message, args=(message,)).start()
```

## Summary

Redis Pub/Sub message loss occurs when subscribers cannot keep up with message throughput, causing their output buffers to overflow and triggering disconnection. Increase client output buffer limits as a short-term fix, but for guaranteed delivery, migrate to Redis Streams with consumer groups. Always monitor subscriber count and client buffer usage in production.
