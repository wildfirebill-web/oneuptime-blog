# How to Use Lua Math Functions in Redis Scripts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Lua, Scripting, Math, Calculation

Description: Learn how to use Lua's math library in Redis scripts for floor, ceil, min, max, and power calculations in server-side data processing.

---

Lua's `math` library provides standard mathematical functions that are useful when processing numeric data server-side in Redis scripts. Using math functions inside scripts avoids round-trips to the application for calculations.

## Core Math Functions

### Rounding

```lua
math.floor(x)  -- Round down to nearest integer
math.ceil(x)   -- Round up to nearest integer

-- Examples
return math.floor(3.7)   -- 3
return math.ceil(3.2)    -- 4
return math.floor(-1.5)  -- -2
return math.ceil(-1.5)   -- -1
```

### Min and Max

```lua
return math.min(10, 5, 8)   -- 5
return math.max(10, 5, 8)   -- 10

-- Clamp a value within bounds
local function clamp(val, min_val, max_val)
    return math.min(math.max(val, min_val), max_val)
end
return clamp(150, 0, 100)   -- 100
```

### Power and Square Root

```lua
return math.sqrt(16)       -- 4.0
return math.pow(2, 10)     -- 1024.0 (Lua 5.1)
return 2 ^ 10              -- 1024.0 (preferred in Redis/LuaJIT)
```

### Absolute Value and Modulo

```lua
return math.abs(-42)      -- 42
return math.fmod(10, 3)   -- 1.0 (floating point modulo)
return 10 % 3             -- 1 (integer modulo, preferred)
```

## Practical Example - Exponential Backoff

Calculate retry delay with exponential backoff:

```lua
-- Returns delay in milliseconds for the nth retry
local retry_count = tonumber(ARGV[1])
local base_delay_ms = tonumber(ARGV[2]) or 100
local max_delay_ms = tonumber(ARGV[3]) or 30000

local delay = base_delay_ms * (2 ^ retry_count)
delay = math.min(delay, max_delay_ms)

-- Store the current backoff state
redis.call('HSET', KEYS[1], 'retries', retry_count, 'next_delay_ms', delay)
return math.floor(delay)
```

```bash
redis-cli EVAL "$(cat backoff.lua)" 1 "job:retry:42" 3 100 30000
# After 3 retries with 100ms base: min(100*8, 30000) = 800
```

## Scoring and Normalization

Normalize a score to a 0-100 scale:

```lua
local raw = tonumber(redis.call('GET', KEYS[1])) or 0
local min_val = tonumber(ARGV[1])
local max_val = tonumber(ARGV[2])

if max_val == min_val then
    return 50  -- Default when range is zero
end

local normalized = (raw - min_val) / (max_val - min_val) * 100
return math.floor(normalized + 0.5)  -- Rounded
```

## Percentile Bucket Calculation

Assign values to percentile buckets:

```lua
local score = tonumber(redis.call('ZSCORE', KEYS[1], ARGV[1])) or 0
local total = tonumber(redis.call('ZCARD', KEYS[1])) or 1
local rank = tonumber(redis.call('ZRANK', KEYS[1], ARGV[1])) or 0

-- Percentile (0-100)
local percentile = math.floor((rank / total) * 100)
return percentile
```

## Important: No Random in Scripts

`math.random()` is available but blocked in Redis scripts to ensure determinism:

```bash
redis-cli EVAL "return math.random()" 0
# ERR This Redis command is not allowed from script
```

For randomness, use `redis.call('RANDOMKEY')` or generate random values before passing them as arguments.

## Math Constants

```lua
return math.pi    -- 3.14159...
return math.huge  -- infinity
return math.maxinteger  -- max integer (Lua 5.3+)
```

## Summary

Lua's `math` library in Redis scripts covers rounding (`floor`, `ceil`), comparisons (`min`, `max`), powers (`^` operator), and absolute values. Use it for server-side calculations like exponential backoff, score normalization, and percentile ranking that would otherwise require an extra network round-trip. Avoid `math.random()` inside scripts as it is blocked to maintain script determinism.
