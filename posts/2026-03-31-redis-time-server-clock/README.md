# How to Use TIME in Redis to Get Server Time

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Time, Server Clock, Unix Timestamp, Scripting

Description: Learn how to use the TIME command in Redis to get the server's current Unix timestamp with microsecond precision for scripts and debugging.

---

## What Is the TIME Command?

The `TIME` command returns the current time from the Redis server's clock. It returns two values:

1. The Unix timestamp in seconds
2. The number of microseconds in the current second

This is useful when you need a consistent timestamp from the Redis server itself rather than relying on client-side clocks, which may differ across machines.

## Basic Usage

```bash
redis-cli TIME
```

Example output:

```text
1) "1711872000"
2) "524813"
```

This means the current time is Unix timestamp `1711872000` with `524813` microseconds elapsed in that second.

## Interpreting the Output

To get the full timestamp in microseconds:

```text
full_microseconds = (seconds * 1_000_000) + microseconds
                  = (1711872000 * 1000000) + 524813
                  = 1711872000524813
```

## Using TIME in Lua Scripts

`TIME` is commonly used inside Lua scripts when you need a server-side timestamp:

```lua
local t = redis.call('TIME')
local seconds = tonumber(t[1])
local micros = tonumber(t[2])
local ms_timestamp = seconds * 1000 + math.floor(micros / 1000)
return ms_timestamp
```

## Why Use Server Time Instead of Client Time?

In distributed systems, clocks can drift across machines. If you generate timestamps on the client, two servers may produce slightly different values. Using `TIME` ensures all timestamps are sourced from the same Redis server clock.

Common use cases:

- Setting expiration windows relative to server time
- Generating monotonic event IDs inside Lua scripts
- Debugging when a key was last written

## Checking Server Clock Drift

You can compare client time to Redis server time to detect drift:

```bash
# Get Redis server time
redis-cli TIME

# Compare with local time
date +%s
```

If the values differ by more than a few seconds, your Redis host may have NTP synchronization issues.

## Using TIME in Python

```python
import redis

r = redis.Redis(host='localhost', port=6379)
seconds, microseconds = r.time()
print(f"Redis server time: {seconds}.{microseconds:06d}")
```

## Restrictions

- `TIME` is read-only with no arguments
- Available since Redis 2.6
- The microseconds value is 0 to 999999

## Summary

`TIME` provides a simple way to get the Redis server's current clock with microsecond precision. It is especially valuable in Lua scripts and distributed scenarios where consistent server-side timestamps matter more than client-generated values.
