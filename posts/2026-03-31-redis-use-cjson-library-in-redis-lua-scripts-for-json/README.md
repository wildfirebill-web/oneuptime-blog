# How to Use cjson Library in Redis Lua Scripts for JSON

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Lua, Scripting, JSON, cjson

Description: Learn to encode and decode JSON data in Redis Lua scripts using the built-in cjson library for structured data storage and retrieval.

---

Redis bundles the `cjson` library in its Lua environment, giving you JSON encoding and decoding inside scripts. This lets you store structured objects as JSON strings in Redis and parse them server-side without extra round-trips.

## cjson Basics

Two primary functions: `cjson.encode()` and `cjson.decode()`:

```lua
-- Encode a Lua table to JSON string
local data = {name = "Alice", age = 30, active = true}
local json_str = cjson.encode(data)
return json_str
-- Returns: {"name":"Alice","age":30,"active":true}
```

```lua
-- Decode a JSON string to Lua table
local json_str = ARGV[1]
local data = cjson.decode(json_str)
return data.name
```

## Storing JSON Objects in Redis

```lua
-- Store a user profile as JSON
local profile = {
    id = ARGV[1],
    name = ARGV[2],
    email = ARGV[3],
    created_at = tonumber(redis.call('TIME')[1])
}

local json_str = cjson.encode(profile)
redis.call('SET', KEYS[1], json_str)
redis.call('EXPIRE', KEYS[1], 86400)

return 'OK'
```

```bash
redis-cli EVAL "$(cat store_profile.lua)" 1 "user:42" "42" "Alice" "alice@example.com"
```

## Reading and Updating JSON Fields

Update a single field in a stored JSON object atomically:

```lua
-- Update a field in a JSON document without overwriting the whole object
local raw = redis.call('GET', KEYS[1])
if raw == false then
    return redis.error_reply('Key not found')
end

local doc = cjson.decode(raw)
doc[ARGV[1]] = ARGV[2]  -- Update field

redis.call('SET', KEYS[1], cjson.encode(doc))
return 'OK'
```

```bash
redis-cli EVAL "$(cat update_field.lua)" 1 "user:42" "name" "Bob"
```

## Working with JSON Arrays

```lua
-- Append an item to a JSON array stored in Redis
local raw = redis.call('GET', KEYS[1])
local list = {}

if raw ~= false then
    list = cjson.decode(raw)
end

table.insert(list, ARGV[1])

-- Keep only last N items
local max_items = tonumber(ARGV[2]) or 100
while #list > max_items do
    table.remove(list, 1)
end

redis.call('SET', KEYS[1], cjson.encode(list))
return #list
```

## Handling cjson Errors

Malformed JSON will cause `cjson.decode` to throw a Lua error. Catch it with `pcall`:

```lua
local raw = redis.call('GET', KEYS[1])
if raw == false then
    return redis.error_reply('Key not found')
end

local ok, result = pcall(cjson.decode, raw)
if not ok then
    return redis.error_reply('Invalid JSON in key: ' .. KEYS[1])
end

return result.field_name or 'null'
```

## JSON Null Values

`cjson` maps JSON `null` to `cjson.null` (not Lua `nil`):

```lua
local data = cjson.decode('{"name": "Alice", "phone": null}')

if data.phone == cjson.null then
    return 'Phone number not set'
end
return data.phone
```

## Performance Tip

Decoding and encoding JSON on every operation adds overhead. For frequently updated individual fields, consider using Redis Hashes instead of JSON strings:

```bash
# Hash approach - more efficient for field-level updates
redis-cli HSET user:42 name "Alice" email "alice@example.com"
redis-cli HGET user:42 name
```

Reserve JSON storage for complex nested structures or when you need atomic multi-field reads.

## Summary

The `cjson` library in Redis Lua scripts provides `cjson.encode()` to serialize Lua tables to JSON strings and `cjson.decode()` to parse JSON strings into Lua tables. Use `pcall` to catch parsing errors from malformed JSON. Remember that `cjson` maps JSON `null` to `cjson.null`, not Lua `nil`. For simple key-value structures, Redis Hashes are more efficient than JSON strings.
