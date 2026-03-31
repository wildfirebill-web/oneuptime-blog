# How to Use cmsgpack Library in Redis Lua Scripts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Lua, Scripting, MessagePack, Serialization

Description: Learn to use the cmsgpack library in Redis Lua scripts to encode and decode MessagePack binary data for compact, efficient data storage.

---

Redis bundles the `cmsgpack` library in its Lua environment, enabling MessagePack serialization inside scripts. MessagePack is a binary serialization format that is faster to encode/decode and more compact than JSON, making it ideal for high-throughput Redis workloads.

## MessagePack vs JSON

MessagePack advantages:
- More compact binary format (typically 20-40% smaller than JSON)
- Faster encoding and decoding
- Supports binary data natively

Trade-off: binary data is not human-readable in `redis-cli`.

## Basic Usage

```lua
-- Encode a Lua table to MessagePack bytes
local data = {name = "Alice", score = 100, tags = {"redis", "lua"}}
local packed = cmsgpack.pack(data)

redis.call('SET', KEYS[1], packed)
return #packed  -- Returns byte size
```

```lua
-- Decode MessagePack bytes back to Lua table
local packed = redis.call('GET', KEYS[1])
if packed == false then
    return redis.error_reply('Key not found')
end

local data = cmsgpack.unpack(packed)
return data.name  -- "Alice"
```

## Storing Compact Session Data

MessagePack is well-suited for session data where storage efficiency matters:

```lua
-- Store session as MessagePack
local session = {
    user_id = tonumber(ARGV[1]),
    roles = {},
    expires = tonumber(redis.call('TIME')[1]) + 3600
}

-- Parse roles from comma-separated ARGV[2]
for role in string.gmatch(ARGV[2], '[^,]+') do
    table.insert(session.roles, role)
end

local packed = cmsgpack.pack(session)
redis.call('SET', KEYS[1], packed)
redis.call('EXPIRE', KEYS[1], 3600)
return #packed  -- Return serialized size in bytes
```

```bash
redis-cli EVAL "$(cat store_session.lua)" 1 "session:abc123" "42" "admin,editor"
```

## Batch Data Storage

Store and retrieve arrays of objects efficiently:

```lua
-- Pack an array of records from ARGV
local records = {}
local i = 1
while i <= #ARGV do
    table.insert(records, {
        id = ARGV[i],
        value = tonumber(ARGV[i+1]) or 0
    })
    i = i + 2
end

local packed = cmsgpack.pack(records)
redis.call('SET', KEYS[1], packed)
return #records
```

## Unpacking and Querying Fields

```lua
-- Retrieve a specific field from a MessagePack-encoded object
local packed = redis.call('GET', KEYS[1])
if packed == false then
    return false
end

local data = cmsgpack.unpack(packed)
local field = ARGV[1]

if data[field] == nil then
    return false
end

return tostring(data[field])
```

## Handling cmsgpack Errors

Protect against corrupt or unexpected data:

```lua
local raw = redis.call('GET', KEYS[1])
if raw == false then
    return redis.error_reply('Key not found')
end

local ok, data = pcall(cmsgpack.unpack, raw)
if not ok then
    return redis.error_reply('Failed to decode MessagePack data')
end

return tostring(data.score or 0)
```

## Compare Storage Size

```lua
-- Compare JSON vs MessagePack for the same data
local data = {id = 1, name = "test", value = 42.5, active = true}

local json_encoded = cjson.encode(data)
local msgpack_encoded = cmsgpack.pack(data)

return {
    tostring(#json_encoded),       -- JSON byte size
    tostring(#msgpack_encoded)     -- MessagePack byte size
}
```

Typically MessagePack is 20-40% smaller, which matters when storing millions of objects.

## When to Use cmsgpack vs cjson

Use `cmsgpack` when:
- Storage efficiency is critical (millions of keys)
- Data is only accessed via Lua scripts or clients with MessagePack support
- You need to store binary data (images, binary IDs)

Use `cjson` when:
- You need human-readable data in `redis-cli`
- Clients expect JSON format
- Debugging and observability matter more than storage efficiency

## Summary

The `cmsgpack` library in Redis Lua scripts provides `cmsgpack.pack()` to serialize Lua tables to compact binary MessagePack format and `cmsgpack.unpack()` to deserialize them. MessagePack is typically 20-40% smaller than JSON and faster to process, making it ideal for high-volume session storage and caching. Use `pcall` to safely handle corrupt or unexpected binary data.
