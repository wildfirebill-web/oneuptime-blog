# How to Log Messages from Redis Lua Scripts (redis.log)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Lua, Scripting, Logging, Debugging

Description: Learn how to use redis.log in Lua scripts to write debug, verbose, notice, and warning messages to the Redis server log for troubleshooting.

---

When a Redis Lua script misbehaves in production, you need a way to trace what's happening. `redis.log()` writes messages directly to the Redis server log, giving you visibility into script execution without modifying return values.

## redis.log Syntax

```lua
redis.log(log_level, message)
```

Log levels available:
- `redis.LOG_DEBUG` - Most verbose, only shown when loglevel is debug
- `redis.LOG_VERBOSE` - Shown at verbose and above
- `redis.LOG_NOTICE` - Shown at notice and above (default level)
- `redis.LOG_WARNING` - Always shown

## Basic Logging Example

```lua
redis.log(redis.LOG_NOTICE, "Script started")

local val = redis.call('GET', KEYS[1])
redis.log(redis.LOG_DEBUG, "Got value: " .. tostring(val))

if val == false then
    redis.log(redis.LOG_WARNING, "Key not found: " .. KEYS[1])
    return 0
end

return tonumber(val)
```

Check Redis logs to see output:

```bash
# View Redis log file
tail -f /var/log/redis/redis.log

# Or via journalctl
journalctl -u redis -f
```

## Set Redis Log Level

To see DEBUG and VERBOSE messages:

```bash
redis-cli CONFIG SET loglevel debug
redis-cli CONFIG SET loglevel verbose
```

Reset to default:

```bash
redis-cli CONFIG SET loglevel notice
```

## Logging Argument Values

Log input validation:

```lua
redis.log(redis.LOG_DEBUG, "KEYS count: " .. tostring(#KEYS))
redis.log(redis.LOG_DEBUG, "ARGV count: " .. tostring(#ARGV))

for i = 1, #KEYS do
    redis.log(redis.LOG_DEBUG, "KEYS[" .. i .. "] = " .. tostring(KEYS[i]))
end
for i = 1, #ARGV do
    redis.log(redis.LOG_DEBUG, "ARGV[" .. i .. "] = " .. tostring(ARGV[i]))
end
```

## Logging Tables

Lua tables cannot be directly logged - convert to string first:

```lua
-- Option 1: log individual fields
local data = {name = "Alice", score = 100}
redis.log(redis.LOG_DEBUG, "name=" .. data.name .. " score=" .. tostring(data.score))

-- Option 2: use cjson to serialize
redis.log(redis.LOG_DEBUG, "data=" .. cjson.encode(data))
```

## Production Logging Pattern

Use different log levels appropriately in production:

```lua
local function process_item(key, value)
    redis.log(redis.LOG_DEBUG, "Processing: " .. key)

    if value == false then
        redis.log(redis.LOG_NOTICE, "Missing key: " .. key)
        return 0
    end

    local n = tonumber(value)
    if n == nil then
        redis.log(redis.LOG_WARNING, "Non-numeric value at " .. key .. ": " .. value)
        return redis.error_reply("Invalid value type")
    end

    return n
end

return process_item(KEYS[1], redis.call('GET', KEYS[1]))
```

## Log Error Conditions

Log before returning errors so you can correlate issues in the log:

```lua
local ttl = tonumber(ARGV[1])
if ttl == nil or ttl <= 0 then
    redis.log(redis.LOG_WARNING,
        "Invalid TTL in script call: " .. tostring(ARGV[1]) ..
        " for key: " .. KEYS[1])
    return redis.error_reply("TTL must be a positive integer")
end

redis.call('EXPIRE', KEYS[1], ttl)
return 1
```

## Performance Impact of Logging

`redis.log()` calls are synchronous and write to the log file. Excessive debug logging in high-throughput scripts can add latency:

- Use `redis.LOG_DEBUG` for development only
- Remove or reduce logging before deploying to production
- Reserve `redis.LOG_WARNING` for genuine error conditions

## Summary

`redis.log()` writes messages to the Redis server log at four levels: DEBUG, VERBOSE, NOTICE, and WARNING. Use it to trace script execution, log argument values, and capture error conditions. Set `loglevel verbose` or `loglevel debug` temporarily to see lower-level messages. In production, log only at NOTICE or WARNING to avoid performance overhead.
