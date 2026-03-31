# How to Use Upstash REST API for Serverless Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Upstash, Serverless, REST API, Edge Computing

Description: Use the Upstash REST API to access Redis over HTTP from serverless and edge runtimes that cannot use TCP connections, with zero connection management.

---

## What Is Upstash REST API

Upstash provides a fully managed Redis service with an HTTP REST API alongside the standard Redis protocol. This makes it usable in environments that block TCP connections, including:

- Cloudflare Workers
- Vercel Edge Functions
- Deno Deploy
- Netlify Edge Functions
- Any HTTP-only runtime

## Authentication and Base URL

Every Upstash database has a REST endpoint and a bearer token:

```bash
# Base URL format
https://<database-name>.upstash.io

# Authenticate using the Authorization header
curl https://your-db.upstash.io/get/mykey \
  -H "Authorization: Bearer your-token-here"
```

## REST API Command Syntax

Commands follow the pattern `/<COMMAND>/<arg1>/<arg2>`:

```bash
# SET a key
curl -X POST https://your-db.upstash.io/set/name/Alice \
  -H "Authorization: Bearer $TOKEN"

# GET a key
curl https://your-db.upstash.io/get/name \
  -H "Authorization: Bearer $TOKEN"

# SETEX with TTL
curl -X POST https://your-db.upstash.io/setex/session/3600/data \
  -H "Authorization: Bearer $TOKEN"

# INCR
curl -X POST https://your-db.upstash.io/incr/pageviews \
  -H "Authorization: Bearer $TOKEN"
```

Response format:

```json
{"result": "Alice"}
```

## Using the Official SDK

The `@upstash/redis` SDK wraps the REST API and supports multiple runtimes:

```bash
npm install @upstash/redis
```

```javascript
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL,
  token: process.env.UPSTASH_REDIS_REST_TOKEN,
});

// All standard Redis commands work
await redis.set('greeting', 'Hello World');
const value = await redis.get('greeting');
console.log(value); // "Hello World"

// With TTL
await redis.setex('session:abc123', 3600, JSON.stringify({ userId: 42 }));

// Hashes
await redis.hset('user:1', { name: 'Alice', email: 'alice@example.com' });
const user = await redis.hgetall('user:1');
```

## Pipeline for Batch Commands

Pipelines reduce HTTP round trips by batching commands into a single request:

```javascript
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL,
  token: process.env.UPSTASH_REDIS_REST_TOKEN,
});

// Execute multiple commands in one HTTP call
const pipeline = redis.pipeline();
pipeline.set('key1', 'value1');
pipeline.set('key2', 'value2');
pipeline.get('key1');
pipeline.incr('counter');

const results = await pipeline.exec();
// results = ['OK', 'OK', 'value1', 1]
```

## Using the Raw HTTP API Directly

For runtimes without SDK support, use the REST API directly:

```javascript
async function redisGet(url, token, key) {
  const response = await fetch(`${url}/get/${encodeURIComponent(key)}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  const data = await response.json();
  return data.result;
}

async function redisSet(url, token, key, value, ttl) {
  const endpoint = ttl
    ? `${url}/setex/${encodeURIComponent(key)}/${ttl}/${encodeURIComponent(value)}`
    : `${url}/set/${encodeURIComponent(key)}/${encodeURIComponent(value)}`;

  const response = await fetch(endpoint, {
    method: 'POST',
    headers: { Authorization: `Bearer ${token}` },
  });
  return response.json();
}

// Usage in Cloudflare Worker
export default {
  async fetch(request, env) {
    const value = await redisGet(
      env.UPSTASH_URL,
      env.UPSTASH_TOKEN,
      'config'
    );
    return new Response(value);
  },
};
```

## Working with Complex Data Types

```javascript
import { Redis } from '@upstash/redis';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL,
  token: process.env.UPSTASH_REDIS_REST_TOKEN,
});

// Sorted sets for leaderboards
await redis.zadd('leaderboard', { score: 1500, member: 'alice' });
await redis.zadd('leaderboard', { score: 2000, member: 'bob' });
const topPlayers = await redis.zrange('leaderboard', 0, 9, { rev: true, withScores: true });

// Lists for queues
await redis.rpush('job-queue', JSON.stringify({ id: 1, task: 'email' }));
const job = await redis.lpop('job-queue');

// Sets for unique tracking
await redis.sadd('visitors:today', 'user:123');
const isNew = await redis.sismember('visitors:today', 'user:123');
```

## Rate Limiting with @upstash/ratelimit

```javascript
import { Redis } from '@upstash/redis';
import { Ratelimit } from '@upstash/ratelimit';

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL,
  token: process.env.UPSTASH_REDIS_REST_TOKEN,
});

const ratelimit = new Ratelimit({
  redis,
  limiter: Ratelimit.slidingWindow(20, '1 m'),
});

const { success } = await ratelimit.limit('user:42');
if (!success) {
  throw new Error('Rate limit exceeded');
}
```

## Summary

The Upstash REST API brings Redis functionality to any HTTP-capable runtime, solving the TCP connection limitation in serverless and edge environments. Use the `@upstash/redis` SDK for a familiar command-style API with automatic serialization, and leverage pipelines to batch multiple operations into a single HTTP request for optimal performance.
