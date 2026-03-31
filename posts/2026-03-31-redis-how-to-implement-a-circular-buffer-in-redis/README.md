# How to Implement a Circular Buffer in Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Circular Buffer, Data Structures, Lists, Lua, Backend

Description: Learn how to implement a fixed-size circular buffer in Redis using Lists and Lua scripts for efficient bounded data storage with automatic overwrite.

---

## What Is a Circular Buffer?

A circular buffer (also called a ring buffer) is a fixed-size queue where new data overwrites the oldest data when the buffer is full. It is ideal for storing sliding windows of recent events, logs, or metrics without unbounded memory growth.

## Why Use Redis for a Circular Buffer?

Redis Lists support O(1) push and pop operations and the LTRIM command to enforce size limits. Unlike application-level implementations, Redis provides atomicity and persistence.

## Basic Implementation with RPUSH and LTRIM

The simplest approach: push to the right, trim from the left to maintain max size.

```bash
# Push an item and trim to last 5 elements
RPUSH buffer:logs "event-1"
LTRIM buffer:logs -5 -1

RPUSH buffer:logs "event-2"
LTRIM buffer:logs -5 -1

# Read all items in order
LRANGE buffer:logs 0 -1
```

## Node.js Implementation

```javascript
const Redis = require('ioredis');
const redis = new Redis({ host: 'localhost', port: 6379 });

class CircularBuffer {
  constructor(key, maxSize) {
    this.key = key;
    this.maxSize = maxSize;
  }

  async push(value) {
    const serialized = typeof value === 'object' ? JSON.stringify(value) : String(value);
    const pipeline = redis.pipeline();
    pipeline.rpush(this.key, serialized);
    pipeline.ltrim(this.key, -this.maxSize, -1);
    await pipeline.exec();
  }

  async getAll() {
    const items = await redis.lrange(this.key, 0, -1);
    return items.map(item => {
      try { return JSON.parse(item); } catch { return item; }
    });
  }

  async getLatest(n) {
    const items = await redis.lrange(this.key, -n, -1);
    return items.map(item => {
      try { return JSON.parse(item); } catch { return item; }
    });
  }

  async size() {
    return redis.llen(this.key);
  }

  async clear() {
    return redis.del(this.key);
  }
}

// Usage
const logBuffer = new CircularBuffer('app:recent-logs', 100);

await logBuffer.push({ level: 'error', message: 'Connection timeout', ts: Date.now() });
await logBuffer.push({ level: 'info', message: 'User logged in', ts: Date.now() });

const recentLogs = await logBuffer.getLatest(10);
console.log(recentLogs);
```

## Atomic Push with Lua Script

For strict atomicity (push + trim as one operation), use a Lua script:

```javascript
const circularPushScript = `
local key = KEYS[1]
local maxSize = tonumber(ARGV[1])
local value = ARGV[2]

redis.call('RPUSH', key, value)
redis.call('LTRIM', key, -maxSize, -1)
return redis.call('LLEN', key)
`;

async function atomicPush(key, value, maxSize) {
  const serialized = JSON.stringify(value);
  const newSize = await redis.eval(circularPushScript, 1, key, maxSize, serialized);
  return newSize;
}

// Usage
await atomicPush('metrics:cpu', { value: 85.2, ts: Date.now() }, 300);
```

## Python Implementation

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

class CircularBuffer:
    def __init__(self, key: str, max_size: int):
        self.key = key
        self.max_size = max_size

    def push(self, value):
        serialized = json.dumps(value) if isinstance(value, (dict, list)) else str(value)
        pipe = r.pipeline()
        pipe.rpush(self.key, serialized)
        pipe.ltrim(self.key, -self.max_size, -1)
        pipe.execute()

    def get_all(self):
        items = r.lrange(self.key, 0, -1)
        result = []
        for item in items:
            try:
                result.append(json.loads(item))
            except (json.JSONDecodeError, ValueError):
                result.append(item)
        return result

    def get_latest(self, n: int):
        items = r.lrange(self.key, -n, -1)
        return [json.loads(i) for i in items]

    def size(self) -> int:
        return r.llen(self.key)

# Usage
buf = CircularBuffer('sensor:temperature', 60)
buf.push({"value": 22.5, "unit": "C", "ts": 1711900000})
print(buf.get_all())
```

## Use Case: Rolling Window Metrics

```javascript
async function recordMetric(service, metric, value) {
  const buffer = new CircularBuffer(`metrics:${service}:${metric}`, 60);
  await buffer.push({ value, ts: Date.now() });
}

async function getAverageMetric(service, metric, lastN = 10) {
  const buffer = new CircularBuffer(`metrics:${service}:${metric}`, 60);
  const samples = await buffer.getLatest(lastN);
  if (samples.length === 0) return null;

  const sum = samples.reduce((acc, s) => acc + s.value, 0);
  return sum / samples.length;
}

// Record CPU usage every second
setInterval(async () => {
  const cpu = process.cpuUsage();
  await recordMetric('api-server', 'cpu', cpu.user / 1000);
}, 1000);
```

## Use Case: Recent Events Log

```bash
# Store last 1000 events
RPUSH events:audit '{"user":"alice","action":"login","ts":1711900000}'
LTRIM events:audit -1000 -1

# Get the 20 most recent events
LRANGE events:audit -20 -1

# Count total stored events
LLEN events:audit
```

## Summary

Redis Lists make circular buffer implementation straightforward using RPUSH and LTRIM in a pipeline or Lua script for atomicity. This pattern is ideal for fixed-size logs, rolling metric windows, and recent-event stores. The buffer automatically discards the oldest entries, keeping memory usage bounded without manual cleanup logic.
