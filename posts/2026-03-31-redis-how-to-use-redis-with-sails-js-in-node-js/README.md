# How to Use Redis with Sails.js in Node.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Sails.js, Node.js, Session, Caching

Description: Learn how to configure Redis in a Sails.js application for sessions, caching, and socket scaling with practical setup examples.

---

Sails.js is a full-featured MVC framework for Node.js built on Express. Redis integrates naturally with Sails for session storage, key-value caching, and scaling Socket.io across multiple processes. This guide covers the most common use cases.

## Installing Dependencies

```bash
npm install ioredis connect-redis @sailshq/socket.io-redis
```

## Configuring Redis Sessions

Sails uses Express-compatible session middleware. Replace the default in-memory session store with Redis.

Edit `config/session.js`:

```javascript
const Redis = require('ioredis');
const RedisStore = require('connect-redis').default;

const redisClient = new Redis({
  host: process.env.REDIS_HOST || '127.0.0.1',
  port: 6379,
});

module.exports.session = {
  secret: process.env.SESSION_SECRET || 'change-me',
  store: new RedisStore({ client: redisClient }),
  cookie: {
    maxAge: 24 * 60 * 60 * 1000, // 1 day
    secure: process.env.NODE_ENV === 'production',
  },
};
```

## Manual Caching in Controllers

You can use `ioredis` directly in Sails controllers for fine-grained caching control.

```javascript
// api/controllers/ProductController.js
const Redis = require('ioredis');
const redis = new Redis();

module.exports = {
  find: async function (req, res) {
    const cacheKey = 'products:all';
    const cached = await redis.get(cacheKey);
    if (cached) {
      return res.json(JSON.parse(cached));
    }
    const products = await Product.find();
    await redis.setex(cacheKey, 120, JSON.stringify(products)); // 2-minute TTL
    return res.json(products);
  },

  invalidate: async function (req, res) {
    await redis.del('products:all');
    return res.ok({ message: 'Cache cleared' });
  },
};
```

## Scaling Socket.io with Redis Adapter

When running multiple Sails.js instances, you need a shared adapter so socket messages are broadcast correctly across all processes.

Edit `config/sockets.js`:

```javascript
const redisAdapter = require('@sailshq/socket.io-redis');

module.exports.sockets = {
  adapter: 'redis',
  adapterModule: redisAdapter,
  host: process.env.REDIS_HOST || '127.0.0.1',
  port: 6379,
};
```

## Using Redis in a Hook

For application-wide Redis access, create a custom Sails hook.

```javascript
// api/hooks/redis/index.js
const Redis = require('ioredis');

module.exports = function defineRedisHook(sails) {
  return {
    initialize: function (cb) {
      sails.redis = new Redis({
        host: sails.config.redis.host || '127.0.0.1',
        port: sails.config.redis.port || 6379,
      });
      sails.redis.on('ready', () => {
        sails.log.info('Redis hook initialized');
        cb();
      });
      sails.redis.on('error', cb);
    },
  };
};
```

Access it in any controller:

```javascript
const value = await sails.redis.get('my-key');
```

## Storing Rate Limit Counters

```javascript
async function rateLimit(userId) {
  const key = `ratelimit:${userId}`;
  const count = await redis.incr(key);
  if (count === 1) await redis.expire(key, 60);
  return count <= 50; // Allow up to 50 requests per minute
}
```

## Summary

Sails.js works with Redis for session storage via `connect-redis`, in-controller caching via `ioredis`, and multi-process socket scaling via the `socket.io-redis` adapter. Wrapping Redis in a custom Sails hook is the cleanest way to share a single connection across your entire application.
