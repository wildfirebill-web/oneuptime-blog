# How to Build a Game Session Manager with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Game Session, Multiplayer, State Management, Backend

Description: Build a game session manager with Redis hashes and sorted sets to track active game states, handle player reconnects, and persist scores with minimal latency.

---

Once a multiplayer game starts, the server needs to track live game state, handle player disconnections, and persist results. Redis keeps all active session data in memory for sub-millisecond reads while sorted sets manage leaderboards.

## Creating a Game Session

When a lobby starts a game, create a session hash with initial state:

```python
import redis
import uuid
import json
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)
SESSION_TTL = 3600  # 1 hour max game length

def create_session(lobby_id: str, players: list, game_mode: str) -> str:
    session_id = str(uuid.uuid4())
    pipe = r.pipeline()
    pipe.hset(f"session:{session_id}", mapping={
        "lobby_id": lobby_id,
        "game_mode": game_mode,
        "status": "active",
        "started_at": str(time.time()),
        "round": "1",
    })
    for player_id in players:
        pipe.hset(f"session:{session_id}:player:{player_id}", mapping={
            "score": "0",
            "connected": "1",
            "last_action_at": str(time.time()),
        })
        pipe.sadd(f"session:{session_id}:players", player_id)
    pipe.expire(f"session:{session_id}", SESSION_TTL)
    pipe.execute()
    return session_id
```

## Updating Player State

```python
def update_score(session_id: str, player_id: str, delta: int):
    key = f"session:{session_id}:player:{player_id}"
    new_score = r.hincrby(key, "score", delta)
    # Update the session leaderboard
    r.zadd(f"session:{session_id}:leaderboard", {player_id: new_score})
    r.hset(key, "last_action_at", str(time.time()))
    return new_score

def record_action(session_id: str, player_id: str, action: dict):
    r.xadd(f"session:{session_id}:actions", {
        "player_id": player_id,
        "action": json.dumps(action),
        "ts": str(time.time()),
    })
```

## Handling Disconnection and Reconnection

```python
DISCONNECT_GRACE_PERIOD = 30  # seconds

def player_disconnected(session_id: str, player_id: str):
    r.hset(f"session:{session_id}:player:{player_id}", "connected", "0")
    # Set a timeout key - if not reconnected, player forfeits
    r.setex(f"session:{session_id}:timeout:{player_id}", DISCONNECT_GRACE_PERIOD, "1")

def player_reconnected(session_id: str, player_id: str):
    r.hset(f"session:{session_id}:player:{player_id}", mapping={
        "connected": "1",
        "last_action_at": str(time.time()),
    })
    r.delete(f"session:{session_id}:timeout:{player_id}")
```

## Getting the Leaderboard

```python
def get_leaderboard(session_id: str) -> list:
    entries = r.zrevrange(f"session:{session_id}:leaderboard", 0, -1, withscores=True)
    return [
        {"rank": i + 1, "player_id": player_id, "score": int(score)}
        for i, (player_id, score) in enumerate(entries)
    ]
```

## Ending a Session

```python
def end_session(session_id: str):
    leaderboard = get_leaderboard(session_id)
    r.hset(f"session:{session_id}", mapping={
        "status": "complete",
        "ended_at": str(time.time()),
        "final_scores": json.dumps(leaderboard),
    })
    # Persist final scores to permanent storage
    persist_session_results(session_id, leaderboard)
```

## Summary

Redis hashes store mutable per-player and per-session state with O(1) read and write. Sorted sets provide a live leaderboard updated on every score change. TTL-based disconnect timeouts handle unclean disconnections gracefully. Streams record the full action history for replay, analytics, or anti-cheat review after the session ends.

