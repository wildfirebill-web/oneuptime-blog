# How to Use Redis with Hapi.js in Node.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Hapi.js, Node.js, Caching, Backend

Description: Learn how to connect Redis to a Hapi.js Node.js application for caching, server-side sessions, and pub/sub messaging with practical examples.

---

Hapi.js is a configuration-driven Node.js framework known for its rich plugin ecosystem. Redis integrates well with Hapi for caching, session management, and pub/sub. This guide shows you how to set up Redis in a Hapi.js project step by step.

## Installing Dependencies

```bash
npm install @hapi/hapi ioredis @hapi/catbox-redis
```

## Connecting Redis with ioredis

```javascript
const Redis = require('ioredis');

const redis = new Redis({
  host: process.env.REDIS_HOST || '127.0.0.1',
  port: 6379,
  password: process.env.REDIS_PASSWORD,
  retryStrategy: (times) => Math.min(times * 100, 3000),
});
```

## Using Hapi's Built-in Cache with Redis

Hapi has a native caching layer called Catbox. You can configure it to use Redis as a backend.

```javascript
const Hapi = require('@hapi/hapi');

const server = Hapi.server({
  port: 3000,
  cache: [
    {
      name: 'redisCache',
      provider: {
        constructor: require('@hapi/catbox-redis'),
        options: {
          host: '127.0.0.1',
          port: 6379,
        },
      },
    },
  ],
});

const productCache = server.cache({
  cache: 'redisCache',
  segment: 'products',
  expiresIn: 60 * 1000, // 60 seconds
  generateFunc: async (id) => {
    // Simulate database fetch
    return { id, name: 'Widget' };
  },
  generateTimeout: 5000,
});

server.route({
  method: 'GET',
  path: '/products/{id}',
  handler: async (request) => {
    return productCache.get(request.params.id);
  },
});
```

## Manual Redis Caching in Route Handlers

For more control, use `ioredis` directly inside route handlers.

```javascript
server.route({
  method: 'GET',
  path: '/users/{id}',
  handler: async (request, h) => {
    const key = `user:${request.params.id}`;
    const cached = await redis.get(key);
    if (cached) {
      return JSON.parse(cached);
    }
    const user = { id: request.params.id, name: 'Alice' }; // Simulate DB fetch
    await redis.setex(key, 300, JSON.stringify(user));
    return user;
  },
});
```

## Server Extension for Redis Lifecycle

Attach Redis startup and shutdown to the Hapi server lifecycle.

```javascript
server.ext('onPreStart', async () => {
  console.log('Verifying Redis connection...');
  await redis.ping();
  console.log('Redis ready');
});

server.ext('onPostStop', async () => {
  await redis.quit();
  console.log('Redis connection closed');
});
```

## Pub/Sub Messaging in Hapi

Use separate Redis client instances for publish and subscribe operations.

```javascript
const publisher = new Redis();
const subscriber = new Redis();

subscriber.subscribe('notifications', (err) => {
  if (err) console.error('Subscribe error:', err);
});

subscriber.on('message', (channel, message) => {
  console.log(`Received on ${channel}:`, message);
});

server.route({
  method: 'POST',
  path: '/notify',
  handler: async (request) => {
    await publisher.publish('notifications', JSON.stringify(request.payload));
    return { sent: true };
  },
});
```

## Summary

Hapi.js integrates with Redis through either the Catbox provider for native server caching or directly via `ioredis` for manual control. Attaching Redis connection management to the Hapi server lifecycle ensures clean startup and graceful shutdown in production.
