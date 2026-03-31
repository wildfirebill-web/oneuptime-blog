# How to Handle Redis Connection Errors in Node.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Node.js, Error Handling, ioredis, Resilience

Description: Learn how to handle Redis connection errors in Node.js with ioredis, including retry strategies, circuit breakers, graceful degradation, and health checks.

---

## Types of Redis Connection Errors

When working with Redis in Node.js, you'll encounter several error categories:

- `ECONNREFUSED` - Redis server not accepting connections
- `ECONNRESET` - Connection dropped by the server
- `ETIMEDOUT` - Command or connection timed out
- `NOAUTH` - Authentication required but not provided
- `WRONGPASS` - Incorrect password
- `CLUSTERDOWN` - Redis Cluster unavailable

## Basic Error Event Handling

```javascript
const Redis = require('ioredis');

const redis = new Redis({
  host: 'localhost',
  port: 6379,
});

redis.on('error', (err) => {
  console.error('Redis error:', err.message);
});

redis.on('connect', () => {
  console.log('Connected to Redis');
});

redis.on('ready', () => {
  console.log('Redis client ready');
});

redis.on('close', () => {
  console.log('Redis connection closed');
});

redis.on('reconnecting', (delay) => {
  console.log(`Reconnecting in ${delay}ms`);
});

redis.on('end', () => {
  console.log('Redis connection ended (no more retries)');
});
```

## Configuring Retry Strategy

```javascript
const Redis = require('ioredis');

const redis = new Redis({
  host: 'localhost',
  port: 6379,
  retryStrategy: (times) => {
    // Return null to stop retrying
    if (times > 10) {
      console.error('Max retries reached, giving up');
      return null;
    }

    // Exponential backoff: 100ms, 200ms, 400ms, ... capped at 30s
    const delay = Math.min(100 * Math.pow(2, times - 1), 30000);
    console.log(`Retry attempt ${times}, next in ${delay}ms`);
    return delay;
  },
  maxRetriesPerRequest: 3,      // Retry individual commands up to 3 times
  connectTimeout: 5000,          // Fail if can't connect in 5s
  commandTimeout: 2000,          // Fail commands taking longer than 2s
  enableReadyCheck: true,        // Wait until Redis is ready
  enableOfflineQueue: true,      // Queue commands while reconnecting
});
```

## Try/Catch for Individual Commands

```javascript
const Redis = require('ioredis');
const redis = new Redis({ enableOfflineQueue: false });

async function safeGet(key) {
  try {
    return await redis.get(key);
  } catch (err) {
    if (err.message.includes('Connection is closed')) {
      console.warn(`Redis unavailable for GET ${key}`);
      return null;
    }
    throw err;
  }
}

async function safeSet(key, value, ttl) {
  try {
    if (ttl) {
      await redis.setex(key, ttl, value);
    } else {
      await redis.set(key, value);
    }
    return true;
  } catch (err) {
    console.error(`Failed to set ${key}:`, err.message);
    return false;
  }
}
```

## Circuit Breaker Pattern

```javascript
const Redis = require('ioredis');

class ResilientRedisClient {
  constructor(options = {}) {
    this.redis = new Redis(options);
    this.isAvailable = true;
    this.failureCount = 0;
    this.failureThreshold = 5;
    this.recoveryTimeout = 30000;
    this.lastFailureTime = null;

    this.redis.on('error', (err) => {
      this.recordFailure(err);
    });

    this.redis.on('ready', () => {
      this.reset();
    });
  }

  recordFailure(err) {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    console.error(`Redis failure ${this.failureCount}/${this.failureThreshold}:`, err.message);

    if (this.failureCount >= this.failureThreshold) {
      this.isAvailable = false;
      console.error('Circuit breaker OPEN - Redis marked unavailable');
    }
  }

  reset() {
    this.failureCount = 0;
    this.isAvailable = true;
    console.log('Circuit breaker CLOSED - Redis available');
  }

  isCircuitOpen() {
    if (!this.isAvailable && this.lastFailureTime) {
      const elapsed = Date.now() - this.lastFailureTime;
      if (elapsed > this.recoveryTimeout) {
        console.log('Circuit breaker testing recovery...');
        this.isAvailable = true; // Allow one test request
        return false;
      }
    }
    return !this.isAvailable;
  }

  async get(key, fallback = null) {
    if (this.isCircuitOpen()) {
      console.warn(`Circuit open: returning fallback for ${key}`);
      return fallback;
    }

    try {
      return await this.redis.get(key);
    } catch (err) {
      this.recordFailure(err);
      return fallback;
    }
  }

  async set(key, value, ttl) {
    if (this.isCircuitOpen()) {
      console.warn(`Circuit open: skip caching ${key}`);
      return false;
    }

    try {
      if (ttl) {
        await this.redis.setex(key, ttl, value);
      } else {
        await this.redis.set(key, value);
      }
      return true;
    } catch (err) {
      this.recordFailure(err);
      return false;
    }
  }
}

const resilientRedis = new ResilientRedisClient({ host: 'localhost', port: 6379 });

const value = await resilientRedis.get('user:1001:profile', '{}');
```

## Graceful Degradation

```javascript
const Redis = require('ioredis');
const redis = new Redis({ enableOfflineQueue: false });

async function getUserProfile(userId) {
  const cacheKey = `user:${userId}:profile`;

  try {
    const cached = await redis.get(cacheKey);
    if (cached) {
      return { source: 'cache', data: JSON.parse(cached) };
    }
  } catch (err) {
    console.warn(`Cache unavailable for user ${userId}, falling back to DB`);
  }

  // Fallback to database
  const data = await fetchFromDatabase(userId);

  // Try to cache the result (non-critical)
  redis.setex(cacheKey, 3600, JSON.stringify(data)).catch(() => {});

  return { source: 'database', data };
}

async function fetchFromDatabase(userId) {
  // Simulated DB fetch
  return { id: userId, name: 'Alice', email: 'alice@example.com' };
}
```

## Health Check Endpoint

```javascript
const express = require('express');
const Redis = require('ioredis');

const app = express();
const redis = new Redis({ connectTimeout: 1000, commandTimeout: 1000 });

app.get('/health', async (req, res) => {
  const health = { status: 'ok', redis: 'unknown' };

  try {
    const start = Date.now();
    await redis.ping();
    health.redis = 'ok';
    health.redisLatencyMs = Date.now() - start;
  } catch (err) {
    health.status = 'degraded';
    health.redis = 'unavailable';
    health.redisError = err.message;
  }

  const statusCode = health.status === 'ok' ? 200 : 503;
  res.status(statusCode).json(health);
});
```

## Summary

Handling Redis connection errors in Node.js with ioredis requires configuring `retryStrategy` for reconnection logic, listening to `error`, `ready`, and `reconnecting` events, and wrapping commands in try/catch for graceful degradation. For production systems, implement a circuit breaker to prevent cascading failures when Redis is unavailable, and always allow the application to serve requests from fallback sources while Redis recovers.
