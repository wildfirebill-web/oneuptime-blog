# How to Build a Live Sports Score Tracker with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Sports, Real-Time

Description: Build a live sports score tracker with Redis using hashes for game state, Pub/Sub for instant score updates, and sorted sets for league standings.

---

Live sports apps must deliver score updates within seconds of game events while serving thousands of concurrent viewers. Redis is a natural fit: hashes store game state, Pub/Sub broadcasts updates instantly, and sorted sets maintain live standings.

## Data Model

```bash
# Game state hash
HSET game:nfl:12345 home_team "Chiefs" away_team "Eagles" home_score 14 away_score 7 quarter 2 clock "08:34" status live

# League standings sorted set (score = points)
ZADD standings:nfl:2025 28 "Chiefs" 21 "Eagles" 14 "Cowboys"
```

## Setting Up

```python
import redis
import json
import time

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

GAME_PREFIX = "game"
STANDINGS_PREFIX = "standings"
```

## Creating a Game

```python
def create_game(sport: str, game_id: str, home_team: str, away_team: str) -> str:
    key = f"{GAME_PREFIX}:{sport}:{game_id}"
    r.hset(key, mapping={
        "home_team": home_team,
        "away_team": away_team,
        "home_score": 0,
        "away_score": 0,
        "status": "scheduled",
        "quarter": 0,
        "clock": "00:00",
        "updated_at": int(time.time())
    })
    return key
```

## Recording a Score Update

```python
UPDATE_SCORE_SCRIPT = """
local game_key = KEYS[1]
local channel = KEYS[2]
local team_field = ARGV[1]
local points = tonumber(ARGV[2])
local clock = ARGV[3]
local now = ARGV[4]

local new_score = redis.call('HINCRBY', game_key, team_field, points)
redis.call('HSET', game_key, 'clock', clock, 'updated_at', now)

local state = redis.call('HGETALL', game_key)
local payload = cjson.encode({event = 'score', team_field = team_field, new_score = new_score, state = state})
redis.call('PUBLISH', channel, payload)
return payload
"""

update_score = r.register_script(UPDATE_SCORE_SCRIPT)

def score_event(sport: str, game_id: str, team: str, points: int, clock: str):
    game_key = f"{GAME_PREFIX}:{sport}:{game_id}"
    channel = f"{GAME_PREFIX}:{sport}:{game_id}:updates"
    # team is 'home' or 'away'
    team_field = f"{team}_score"

    result = update_score(
        keys=[game_key, channel],
        args=[team_field, points, clock, int(time.time())]
    )
    return json.loads(result)
```

## Subscribing to a Game Feed

```python
import threading

def watch_game(sport: str, game_id: str):
    sub = r.pubsub()
    channel = f"{GAME_PREFIX}:{sport}:{game_id}:updates"
    sub.subscribe(channel)

    print(f"Live updates for game {game_id}:")
    for message in sub.listen():
        if message["type"] != "message":
            continue
        data = json.loads(message["data"])
        state = {data["state"][i]: data["state"][i+1] for i in range(0, len(data["state"]), 2)}
        print(f"  {state['home_team']} {state['home_score']} - "
              f"{state['away_score']} {state['away_team']}  "
              f"Q{state['quarter']} {state['clock']}")
```

## Updating League Standings

```python
def update_standings(sport: str, season: str, game_id: str):
    game_key = f"{GAME_PREFIX}:{sport}:{game_id}"
    state = r.hgetall(game_key)

    home_score = int(state["home_score"])
    away_score = int(state["away_score"])
    standings_key = f"{STANDINGS_PREFIX}:{sport}:{season}"

    if home_score > away_score:
        r.zincrby(standings_key, 2, state["home_team"])
    elif away_score > home_score:
        r.zincrby(standings_key, 2, state["away_team"])
    else:
        r.zincrby(standings_key, 1, state["home_team"])
        r.zincrby(standings_key, 1, state["away_team"])

def get_standings(sport: str, season: str, top_n: int = 10) -> list:
    return r.zrange(
        f"{STANDINGS_PREFIX}:{sport}:{season}",
        0, top_n - 1,
        desc=True,
        withscores=True
    )
```

## Getting Current Score

```python
def get_score(sport: str, game_id: str) -> dict:
    state = r.hgetall(f"{GAME_PREFIX}:{sport}:{game_id}")
    return {
        "home": {"team": state["home_team"], "score": int(state["home_score"])},
        "away": {"team": state["away_team"], "score": int(state["away_score"])},
        "clock": state["clock"],
        "status": state["status"]
    }
```

## Summary

A Redis live sports score tracker uses hashes for mutable game state, Lua scripts for atomic score updates with simultaneous Pub/Sub broadcasts, and sorted sets for real-time standings. This design scales to thousands of subscribers per game without database pressure on every score event.
