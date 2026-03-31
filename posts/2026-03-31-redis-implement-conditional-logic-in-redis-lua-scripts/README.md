# How to Implement Conditional Logic in Redis Lua Scripts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Lua, Scripting, Conditional, Logic

Description: Learn how to write if-then-else conditional logic in Redis Lua scripts to make decisions based on key existence, values, and command results.

---

Conditional logic is what transforms simple Redis commands into powerful server-side operations. With `if/elseif/else` in Lua, your scripts can make decisions based on Redis data and execute different commands accordingly.

## Basic If-Else

```lua
local value = redis.call('GET', KEYS[1])

if value then
    return 'Key exists: ' .. value
else
    return 'Key does not exist'
end
```

## Truthy and Falsy Values in Lua

In Lua, only `false` and `nil` are falsy. Everything else is truthy - including `0` and empty string `""`:

```lua
local val = redis.call('GET', KEYS[1])

-- val is false (Lua false) when key doesn't exist
if val == false then
    return 'Key missing'
end

-- 0 is TRUTHY in Lua
if 0 then
    return 'This always executes'
end

-- Check for actual zero value
local count = tonumber(val) or 0
if count == 0 then
    return 'Counter is zero'
end
```

## If-Elseif-Else Chain

```lua
local action = ARGV[1]
local amount = tonumber(ARGV[2]) or 0

if action == 'add' then
    return redis.call('INCRBY', KEYS[1], amount)
elseif action == 'subtract' then
    return redis.call('DECRBY', KEYS[1], amount)
elseif action == 'reset' then
    return redis.call('SET', KEYS[1], 0)
else
    return redis.error_reply('Unknown action: ' .. action)
end
```

## Set-If-Not-Exists Pattern

A classic use case for conditional logic:

```lua
-- Acquire a distributed lock
local existing = redis.call('GET', KEYS[1])
if existing == false then
    redis.call('SET', KEYS[1], ARGV[1])
    redis.call('EXPIRE', KEYS[1], tonumber(ARGV[2]))
    return 1  -- Lock acquired
else
    return 0  -- Lock already held
end
```

This is equivalent to `SET key value NX EX ttl`, but the script pattern allows more complex logic around the lock.

## Conditional Based on Score

```lua
-- Add points to user score, but cap at maximum
local current_score = tonumber(redis.call('ZSCORE', KEYS[1], ARGV[1])) or 0
local points_to_add = tonumber(ARGV[2])
local max_score = tonumber(ARGV[3]) or 1000

local new_score = current_score + points_to_add

if new_score > max_score then
    new_score = max_score
end

redis.call('ZADD', KEYS[1], new_score, ARGV[1])
return new_score
```

## Short-Circuit with Early Return

Return early to keep scripts readable:

```lua
-- Validate input, then process
if ARGV[1] == nil or ARGV[1] == '' then
    return redis.error_reply('Argument required')
end

local ttl = tonumber(ARGV[2])
if ttl == nil or ttl <= 0 then
    return redis.error_reply('TTL must be a positive integer')
end

redis.call('SET', KEYS[1], ARGV[1])
redis.call('EXPIRE', KEYS[1], ttl)
return 'OK'
```

## Compound Conditions

```lua
local val = tonumber(redis.call('GET', KEYS[1])) or 0
local threshold = tonumber(ARGV[1])
local alert_key = KEYS[2]

-- Compound condition: value is high AND no alert has fired yet
if val >= threshold and redis.call('EXISTS', alert_key) == 0 then
    redis.call('SET', alert_key, '1')
    redis.call('EXPIRE', alert_key, 300)  -- Alert cooldown: 5 minutes
    return 'ALERT'
end

return 'OK'
```

## Summary

Lua conditional logic in Redis scripts uses standard `if/elseif/else/end` syntax. Remember that only `false` and `nil` are falsy - `0` and empty strings are truthy. Check for missing keys with `== false` rather than truthiness checks. Use early returns for input validation to keep scripts clean, and combine conditions with `and`/`or` for complex business rules.
