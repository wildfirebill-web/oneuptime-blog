# How to Build a Daily Quest System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Gaming, Quest, Hash, TTL

Description: Build a daily quest system using Redis hashes and TTL-based resets to track player progress, award rewards, and automatically refresh quests every 24 hours.

---

Daily quests are a core engagement mechanic in games - players return each day to complete objectives and earn rewards. Redis is well suited for this because its TTL feature handles automatic resets and its hash commands efficiently track progress.

## Quest Definition

Store quest templates in Redis hashes:

```bash
HSET quest:template:kill_10_goblins \
  title "Slay 10 Goblins" \
  goal 10 \
  reward_gold 150 \
  reward_xp 200
```

## Assigning Daily Quests

At midnight (or on first login), assign quests to a player and set a 24-hour expiry:

```python
import redis
import time

r = redis.Redis()

def assign_daily_quests(player_id, quest_ids):
    for quest_id in quest_ids:
        key = f"player:{player_id}:quest:{quest_id}:progress"
        r.hset(key, mapping={"progress": 0, "completed": 0, "claimed": 0})
        # Expire at next midnight
        seconds_until_midnight = 86400 - (int(time.time()) % 86400)
        r.expire(key, seconds_until_midnight)
```

## Tracking Progress

When a player kills a goblin, increment their progress:

```bash
HINCRBY player:1001:quest:kill_10_goblins:progress progress 1
```

Check if the quest is now complete:

```python
def check_quest_completion(player_id, quest_id):
    progress_key = f"player:{player_id}:quest:{quest_id}:progress"
    template_key = f"quest:template:{quest_id}"

    progress = int(r.hget(progress_key, "progress") or 0)
    goal = int(r.hget(template_key, "goal") or 0)

    if progress >= goal:
        r.hset(progress_key, "completed", 1)
        return True
    return False
```

## Claiming Rewards

Prevent double-claiming with an atomic check-and-set:

```lua
local key = KEYS[1]
local completed = redis.call("HGET", key, "completed")
local claimed = redis.call("HGET", key, "claimed")
if completed == "1" and claimed == "0" then
  redis.call("HSET", key, "claimed", "1")
  return 1
end
return 0
```

```python
claim_script = r.register_script(open("claim_quest.lua").read())
result = claim_script(keys=[f"player:{player_id}:quest:{quest_id}:progress"])
if result == 1:
    grant_rewards(player_id, quest_id)
```

## Listing Active Quests

Store the player's active quest IDs in a set:

```bash
SADD player:1001:daily_quests kill_10_goblins collect_herbs craft_potion
SMEMBERS player:1001:daily_quests
```

Set this key to expire at the same time as the individual quest progress keys.

## Quest Completion Stats

Track aggregate completion rates across all players:

```bash
INCR stats:quest:kill_10_goblins:started
INCR stats:quest:kill_10_goblins:completed
```

Use these ratios to tune quest difficulty.

## Summary

Redis handles daily quest systems elegantly by combining hash-based progress tracking with TTL-driven automatic resets. Atomic Lua scripts prevent double-reward exploits, while expiry keys eliminate the need for scheduled cleanup jobs - quests simply vanish when the day ends and are re-assigned fresh.
