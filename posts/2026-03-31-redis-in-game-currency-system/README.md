# How to Implement In-Game Currency System with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Gaming, Currency, Cache, Lua

Description: Learn how to build a reliable in-game currency system using Redis atomic operations to handle balances, purchases, and transactions without race conditions.

---

In-game currency (gold, gems, coins) is central to most games. Players earn and spend it constantly, so your backend must handle concurrent transactions without overselling or creating negative balances. Redis is a natural fit because of its atomic commands and Lua scripting support.

## Data Model

Store each player's balance as a Redis hash:

```bash
HSET player:1001:wallet gold 500 gems 20 coins 1000
HGET player:1001:wallet gold
# "500"
```

## Atomic Deduction with Lua

Plain `DECRBY` can go negative. Use a Lua script to check and deduct atomically:

```lua
local key = KEYS[1]
local field = ARGV[1]
local amount = tonumber(ARGV[2])
local balance = tonumber(redis.call("HGET", key, field))
if balance == nil or balance < amount then
  return redis.error_reply("INSUFFICIENT_FUNDS")
end
return redis.call("HINCRBY", key, field, -amount)
```

Run it from your application:

```bash
redis-cli EVAL "$(cat deduct.lua)" 1 player:1001:wallet gold 100
```

## Earning Currency

Add currency on quest completion or purchase:

```bash
HINCRBY player:1001:wallet gold 250
```

## Transaction Logging

Every credit or debit should be logged for auditing and disputes:

```bash
LPUSH player:1001:txlog "earn:gold:250:quest_complete:1679001234"
LTRIM player:1001:txlog 0 999
```

This keeps the last 1,000 transactions per player.

## Handling Purchases

A purchase atomically debits currency and grants an item:

```python
import redis
r = redis.Redis()

def buy_item(player_id, item_id, cost_field, cost_amount):
    wallet_key = f"player:{player_id}:wallet"
    inventory_key = f"player:{player_id}:inventory"
    script = """
    local balance = tonumber(redis.call('HGET', KEYS[1], ARGV[1]))
    if balance == nil or balance < tonumber(ARGV[2]) then
      return redis.error_reply('INSUFFICIENT_FUNDS')
    end
    redis.call('HINCRBY', KEYS[1], ARGV[1], -tonumber(ARGV[2]))
    redis.call('SADD', KEYS[2], ARGV[3])
    return 1
    """
    try:
        r.eval(script, 2, wallet_key, inventory_key, cost_field, cost_amount, item_id)
        return True
    except redis.ResponseError as e:
        return str(e)
```

## Currency Expiry (Time-Limited Events)

For event-specific currencies that expire, use a separate key with TTL:

```bash
SET player:1001:event_coins 100 EX 86400
DECRBY player:1001:event_coins 30
```

## Monitoring

Track economy health with aggregated counters:

```bash
INCR stats:currency:gold:earned
INCR stats:currency:gold:spent
```

Check daily ratios to detect exploits or economy imbalances.

## Summary

Redis gives you atomic, low-latency currency operations via Lua scripting that prevent race conditions and negative balances. Pair balance hashes with transaction logs and aggregate stats to build a complete, auditable in-game economy that scales under high concurrency.
