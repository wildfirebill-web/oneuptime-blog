# What Is New in Redis 7.2 (Triggers, Auto-Tiering)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Trigger, Auto-Tiering, Function, Performance

Description: Redis 7.2 introduced keyspace triggers for event-driven scripting, function enhancements, and the foundation for auto-tiering memory management.

---

Redis 7.2, released in August 2023, built on the Redis Functions platform introduced in 7.0 by adding triggers, expanding function capabilities, and introducing auto-tiering for memory-over-flash storage.

## Keyspace Triggers

Redis 7.2 added the ability to register functions as triggers that fire when specific keyspace events occur. This enables server-side reactive logic without external consumers.

Load a function with a trigger:

```bash
redis-cli FUNCTION LOAD "#!lua name=auditlib

  local function on_key_expire(keys, args)
    local key = keys[1]
    redis.call('LPUSH', 'expired:log', key .. ':' .. redis.call('TIME')[1])
  end

  redis.register_function('on_expire', on_key_expire)
"
```

Register the trigger:

```bash
redis-cli TFUNCTION LOAD "#!lua name=auditlib
  redis.register_function{
    function_name='on_expire',
    callback=function(keys, args)
      redis.call('LPUSH', 'expired:log', keys[1])
    end,
    flags={'no-writes'}
  }
"
```

Triggers fire synchronously within the same transaction as the triggering event, ensuring consistency.

## Function Enhancements

Redis 7.2 improved the Functions API with:

- Better error messages with stack traces
- `redis.log()` for server-side debug logging
- Async functions using coroutines
- Improved `FUNCTION STATS` command

Check function stats:

```bash
redis-cli FUNCTION STATS
# 1) "running_script"
# 2) 0
# 3) "engines"
# 4) 1) "LUA"
#    2) (stats...)
```

## Auto-Tiering (Redis on Flash)

Redis 7.2 laid the groundwork for auto-tiering, a feature that keeps hot data in RAM and automatically moves cold data to NVMe flash storage. This significantly reduces memory costs for large datasets.

Key concepts:
- RAM stores frequently accessed keys (hot data)
- Flash stores infrequently accessed keys (cold data)
- Access patterns determine tier placement automatically

Configuration (Redis Enterprise):

```text
storage-engine flash
flash-path /nvme/redis
ram-size 8gb
flash-size 200gb
```

When a cold key is accessed, it is promoted to RAM. A recently added key that becomes cold is demoted to flash.

## LPOS Improvements

`LPOS` (List Position, added in Redis 6.0.6) received improvements in 7.2 for `RANK` and `MAXLEN` parameters:

```bash
# Find all positions of "target" in list, searching from the tail
redis-cli LPOS mylist target RANK -1 COUNT 0
```

## SINTERCARD - Sorted Set Intersection Count

Redis 7.2 added `LMPOP` with `COUNT` and improved `SINTERCARD` to limit results:

```bash
redis-cli SINTERCARD 2 set1 set2 LIMIT 10
# Returns at most 10 intersecting members
```

## Improved WAIT Command

Redis 7.2 enhanced `WAIT` to support timeouts and return the number of replicas that acknowledged:

```bash
# Wait for 2 replicas to acknowledge with 1 second timeout
redis-cli WAIT 2 1000
# (integer) 2
```

## Summary

Redis 7.2 expanded the Functions platform with keyspace triggers for server-side event-driven logic, improved function debugging, and introduced the auto-tiering foundation for cost-effective large dataset management. These features enable more sophisticated server-side processing patterns without external consumers.
