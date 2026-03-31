# How to Pass Keys and Arguments to Redis Lua Scripts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Lua, Scripting, Key, Argument

Description: Learn how to correctly pass keys and arguments to Redis Lua scripts using KEYS and ARGV arrays, including cluster compatibility best practices.

---

Redis Lua scripts receive input through two special global arrays: `KEYS` for Redis key names and `ARGV` for additional arguments. Understanding how to use these correctly is essential for writing portable, cluster-compatible scripts.

## KEYS vs ARGV

- `KEYS[]` - Redis key names that the script will access. Required for cluster compatibility.
- `ARGV[]` - Non-key arguments like values, counts, or configuration parameters.

Both arrays are 1-indexed in Lua (unlike most programming languages).

## Passing Keys

Keys are specified after the `numkeys` count in `EVAL`:

```bash
# EVAL script numkeys key1 key2 arg1 arg2
redis-cli EVAL "return KEYS[1]" 1 mykey
# Returns: "mykey"

redis-cli EVAL "return {KEYS[1], KEYS[2]}" 2 key1 key2
# Returns: 1) "key1"  2) "key2"
```

A script that copies a value from one key to another:

```bash
redis-cli EVAL "
  local val = redis.call('GET', KEYS[1])
  if val then
    redis.call('SET', KEYS[2], val)
    return 1
  end
  return 0
" 2 source-key dest-key
```

## Passing Arguments

Arguments come after the keys. `ARGV[1]` is the first non-key argument:

```bash
redis-cli EVAL "
  redis.call('SET', KEYS[1], ARGV[1])
  redis.call('EXPIRE', KEYS[1], tonumber(ARGV[2]))
  return 'OK'
" 1 session:user123 "token-data" 3600
```

Here `ARGV[1]` is the value and `ARGV[2]` is the TTL.

## Type Conversion

All arguments arrive as strings in Lua. Convert types explicitly:

```lua
-- Convert string to number
local count = tonumber(ARGV[1])
local threshold = tonumber(ARGV[2])

-- Convert number back to string for Redis
redis.call('SET', KEYS[1], tostring(count + 1))
```

## Dynamic Number of Keys or Arguments

Lua can iterate over KEYS and ARGV using the `#` length operator:

```lua
-- Sum all values at the given keys
local sum = 0
for i = 1, #KEYS do
    local val = tonumber(redis.call('GET', KEYS[i])) or 0
    sum = sum + val
end
return sum
```

```bash
redis-cli EVAL "$(cat sum.lua)" 3 counter1 counter2 counter3
```

## Using Arguments as Configuration

A flexible multi-purpose script:

```bash
redis-cli EVAL "
  local action = ARGV[1]
  if action == 'incr' then
    return redis.call('INCRBY', KEYS[1], tonumber(ARGV[2]))
  elseif action == 'decr' then
    return redis.call('DECRBY', KEYS[1], tonumber(ARGV[2]))
  elseif action == 'reset' then
    return redis.call('SET', KEYS[1], 0)
  end
  return redis.error_reply('Unknown action')
" 1 my-counter incr 5
```

## Cluster Compatibility Rule

In Redis Cluster, all keys accessed by a script must be in the same hash slot. If `KEYS` is declared properly, the cluster client can validate this before sending the script:

```bash
# Both keys must be in the same slot
# Use hash tags {} to force the same slot
redis-cli EVAL "..." 2 "{user:123}:profile" "{user:123}:session"
```

The `{user:123}` hash tag ensures both keys hash to the same slot.

## Summary

Redis Lua scripts receive key names via `KEYS[]` and other arguments via `ARGV[]`, both 1-indexed. Always declare all accessed keys in `KEYS[]` for Redis Cluster compatibility. Convert string arguments to the appropriate Lua types with `tonumber()` and `tostring()`. Use hash tags to co-locate related keys on the same cluster slot when a script must access multiple keys.
