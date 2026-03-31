# How to Use Redis in Your First Web Application

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Web Application, Caching

Description: A beginner-friendly guide to integrating Redis into your first web app for session storage, caching, and fast data retrieval with practical Node.js examples.

---

Redis is a fast, in-memory data store that web developers commonly use to cache responses, store session data, and speed up database-heavy operations. If you are building your first web application, adding Redis is straightforward and pays off quickly in performance.

## Install Redis

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install redis-server -y
sudo systemctl start redis
redis-cli ping
# PONG
```

## Install the Redis Client

```bash
npm install redis express express-session connect-redis
```

## Connect to Redis in Node.js

```javascript
const { createClient } = require('redis');

const client = createClient({ url: 'redis://localhost:6379' });

client.on('error', (err) => console.error('Redis error:', err));

(async () => {
  await client.connect();
  console.log('Connected to Redis');
})();
```

## Cache an API Response

Instead of hitting your database on every request, cache the result in Redis with a TTL (time-to-live).

```javascript
const express = require('express');
const app = express();

app.get('/products', async (req, res) => {
  const cached = await client.get('products');
  if (cached) {
    return res.json({ source: 'cache', data: JSON.parse(cached) });
  }

  // Simulate DB fetch
  const products = [{ id: 1, name: 'Widget' }, { id: 2, name: 'Gadget' }];

  // Store in Redis for 60 seconds
  await client.setEx('products', 60, JSON.stringify(products));

  res.json({ source: 'db', data: products });
});

app.listen(3000, () => console.log('Server on port 3000'));
```

## Store Session Data

Sessions store user login state between requests. Redis is ideal for this because it is fast and supports expiration.

```javascript
const session = require('express-session');
const { RedisStore } = require('connect-redis');

app.use(
  session({
    store: new RedisStore({ client }),
    secret: 'my-secret',
    resave: false,
    saveUninitialized: false,
    cookie: { secure: false, maxAge: 1000 * 60 * 30 }, // 30 minutes
  })
);

app.post('/login', (req, res) => {
  req.session.userId = 42;
  res.json({ message: 'Logged in' });
});

app.get('/profile', (req, res) => {
  if (!req.session.userId) return res.status(401).json({ error: 'Not logged in' });
  res.json({ userId: req.session.userId });
});
```

## Store a Simple Counter

Redis atomic operations make counters safe across concurrent requests.

```javascript
app.post('/track/:page', async (req, res) => {
  const key = `pageview:${req.params.page}`;
  const views = await client.incr(key);
  res.json({ page: req.params.page, views });
});
```

## Verify Data in Redis CLI

```bash
redis-cli
KEYS *
GET products
TTL products
```

## Tips for Beginners

- Always set a TTL on cached keys to avoid stale data filling memory.
- Use namespaced keys like `user:42:profile` to organize data.
- Never store sensitive data (passwords, tokens) in Redis without encryption.
- Use `redis-cli monitor` during development to see every command in real time.

## Summary

Redis adds fast caching and session storage to web applications with minimal setup. By caching database results and storing sessions in Redis, your Node.js app responds faster and scales more easily. Start with simple `setEx` caching, then explore more advanced patterns as your app grows.
