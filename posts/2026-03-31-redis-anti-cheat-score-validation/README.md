# How to Implement Anti-Cheat Score Validation with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Gaming, Anti-Cheat, Security, Lua

Description: Use Redis to implement server-side anti-cheat score validation by tracking rate limits, statistical baselines, and anomaly flags to block inflated scores in real time.

---

Score cheating ruins competitive games. Client-reported scores are inherently untrusted, so your server needs to validate them against expected behavior. Redis enables fast, stateful checks without round-tripping to a database on every score submission.

## Rate Limiting Score Submissions

A legitimate player can only score so fast. Reject submissions that arrive too quickly:

```python
import redis

r = redis.Redis()
MAX_SUBMISSIONS_PER_MINUTE = 30

def rate_limit_score(player_id):
    key = f"anticheat:rate:{player_id}"
    count = r.incr(key)
    if count == 1:
        r.expire(key, 60)
    if count > MAX_SUBMISSIONS_PER_MINUTE:
        return False
    return True
```

## Tracking Score Velocity

Keep a rolling window of scores to detect sudden spikes:

```bash
LPUSH player:1001:score_history 450 380 420 390 410
LTRIM player:1001:score_history 0 19
```

On each new submission, compute the average and flag outliers:

```python
def is_score_suspicious(player_id, new_score, threshold_multiplier=3):
    history = r.lrange(f"player:{player_id}:score_history", 0, -1)
    if len(history) < 5:
        return False
    scores = [int(s) for s in history]
    avg = sum(scores) / len(scores)
    return new_score > avg * threshold_multiplier
```

## Session Token Validation

Issue a signed session token at game start and verify it on score submission:

```bash
SET session:1001:token "abc123xyz" EX 3600
```

```python
def validate_session(player_id, token):
    stored = r.get(f"session:{player_id}:token")
    return stored and stored.decode() == token
```

This ensures scores come from active, server-initiated sessions.

## Flagging and Banning

When suspicious behavior is detected, increment a strike counter:

```lua
local key = KEYS[1]
local strikes = tonumber(redis.call("INCR", key))
redis.call("EXPIRE", key, 86400)
if strikes >= 3 then
  redis.call("SET", "banned:" .. ARGV[1], "1", "EX", "604800")
  return -1
end
return strikes
```

```python
def record_cheat_strike(player_id):
    script = r.register_script(open("strike.lua").read())
    return script(keys=[f"anticheat:strikes:{player_id}"], args=[player_id])
```

## Score Submission Pipeline

Combine all checks in a validation function:

```python
def submit_score(player_id, score, session_token):
    if not validate_session(player_id, session_token):
        return "INVALID_SESSION"
    if r.exists(f"banned:{player_id}"):
        return "BANNED"
    if not rate_limit_score(player_id):
        return "RATE_LIMITED"
    if is_score_suspicious(player_id, score):
        record_cheat_strike(player_id)
        return "SUSPICIOUS"
    r.lpush(f"player:{player_id}:score_history", score)
    r.ltrim(f"player:{player_id}:score_history", 0, 19)
    return "OK"
```

## Summary

Redis enables real-time anti-cheat validation by combining rate limiting, statistical anomaly detection, and session verification in a single low-latency pipeline. Atomic Lua scripts ensure strike counters and ban flags are updated consistently, giving your game a responsive, server-authoritative defense against score manipulation.
