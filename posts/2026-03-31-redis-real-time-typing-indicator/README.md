# How to Build a Real-Time Typing Indicator with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Chat, Real-Time

Description: Implement typing indicators in chat or collaborative apps using Redis keys with short TTLs and Pub/Sub to broadcast who is currently typing.

---

Typing indicators ("Alice is typing...") are a small UX detail that requires surprisingly careful engineering. Each keypress must update a short-lived state, every other participant must be notified instantly, and stale "is typing" states must expire automatically. Redis TTLs and Pub/Sub solve this cleanly.

## How Typing Indicators Work

1. When a user starts typing, set a Redis key with a short TTL (e.g., 5 seconds).
2. As long as the user keeps typing, refresh the TTL.
3. If the user stops, the key expires on its own - no explicit cleanup needed.
4. On every state change, publish to a channel so other participants get notified.

## Setup

```python
import redis
import json
import time
import threading

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

TYPING_TTL = 5          # seconds before "is typing" auto-clears
TYPING_PREFIX = "typing"
CHANNEL_PREFIX = "room"
```

## Recording a Typing Event

```python
def user_typing(room_id: str, user_id: str, username: str):
    typing_key = f"{TYPING_PREFIX}:{room_id}:{user_id}"

    # Check if the key already existed (avoid duplicate start events)
    was_typing = r.exists(typing_key)

    # Refresh TTL each time a keystroke arrives
    r.setex(typing_key, TYPING_TTL, username)

    if not was_typing:
        # New typing event - notify others
        _broadcast_typing_state(room_id)

def user_stopped_typing(room_id: str, user_id: str):
    typing_key = f"{TYPING_PREFIX}:{room_id}:{user_id}"
    deleted = r.delete(typing_key)
    if deleted:
        _broadcast_typing_state(room_id)
```

## Broadcasting Current Typists

```python
def _broadcast_typing_state(room_id: str):
    typists = get_current_typists(room_id)
    r.publish(f"{CHANNEL_PREFIX}:{room_id}:typing", json.dumps({
        "event": "typing_update",
        "typists": typists,
        "ts": int(time.time())
    }))

def get_current_typists(room_id: str) -> list[str]:
    pattern = f"{TYPING_PREFIX}:{room_id}:*"
    keys = r.keys(pattern)
    if not keys:
        return []
    usernames = r.mget(keys)
    return [u for u in usernames if u]
```

## Subscribing to Typing Events

```python
def watch_typing(room_id: str, my_user_id: str):
    sub = r.pubsub()
    sub.subscribe(f"{CHANNEL_PREFIX}:{room_id}:typing")

    for message in sub.listen():
        if message["type"] != "message":
            continue
        data = json.loads(message["data"])
        typists = data["typists"]
        if typists:
            print(f"{', '.join(typists)} {'is' if len(typists) == 1 else 'are'} typing...")
        else:
            print("")
```

## Expiry-Triggered Cleanup with Keyspace Notifications

Enable keyspace notifications so you get notified when a typing key expires:

```bash
redis-cli config set notify-keyspace-events "KEx"
```

```python
def listen_for_expiry():
    sub = r.pubsub()
    sub.psubscribe("__keyevent@0__:expired")

    for message in sub.listen():
        if message["type"] != "pmessage":
            continue
        expired_key = message["data"]
        if expired_key.startswith(f"{TYPING_PREFIX}:"):
            parts = expired_key.split(":")
            if len(parts) == 3:
                room_id = parts[1]
                _broadcast_typing_state(room_id)
```

## Throttling Typing Events on the Client

To avoid flooding Redis with every single keystroke, throttle updates on the sender side:

```python
last_typing_event = {}

def on_keypress(room_id: str, user_id: str, username: str):
    now = time.time()
    last = last_typing_event.get(user_id, 0)

    if now - last > (TYPING_TTL / 2):
        user_typing(room_id, user_id, username)
        last_typing_event[user_id] = now
```

## Summary

Redis typing indicators use short-TTL keys that auto-expire when a user stops typing, Pub/Sub to broadcast the current set of typists to all room participants, and optional keyspace notifications to trigger cleanup events on expiry. Throttling keypress events on the client keeps Redis traffic minimal while maintaining a responsive experience.
