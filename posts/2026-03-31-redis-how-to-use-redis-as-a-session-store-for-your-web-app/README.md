# How to Use Redis as a Session Store for Your Web App

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Session, Web Development, Authentication, Node.js, Security

Description: Learn how to store web application sessions in Redis for fast, scalable authentication across multiple server instances.

---

## Why Store Sessions in Redis

By default, web frameworks store sessions in memory on a single server. This breaks when you scale to multiple servers - a user logged in on server A gets logged out when their next request hits server B. Redis solves this by providing a shared, fast session store accessible by all your servers.

## How Sessions Work

1. User logs in - server creates a session ID and stores session data in Redis
2. Session ID is sent to the browser as a cookie
3. On subsequent requests, the browser sends the cookie
4. Server looks up the session ID in Redis to retrieve user data

```text
Browser           Server          Redis
  |-- POST /login -->|             |
  |                  |-- SETEX --->|  session:abc123 = {userId:42}
  |<-- Set-Cookie ---|             |
  |  (sessionId=abc123)            |
  |                               |
  |-- GET /dashboard cookie:abc123->|
  |                  |-- GET ----->|  session:abc123
  |                  |<-- {userId:42}|
  |<-- 200 OK -------|             |
```

## Setting Up with Node.js and Express

```bash
npm install express express-session connect-redis redis
```

```javascript
const express = require('express');
const session = require('express-session');
const { createClient } = require('redis');
const RedisStore = require('connect-redis').default;

const app = express();

// Create Redis client
const redisClient = createClient({
  url: 'redis://localhost:6379'
});
redisClient.connect();

// Configure session middleware
app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 1000 * 60 * 60 * 24  // 24 hours
  }
}));

// Login route
app.post('/login', async (req, res) => {
  const user = await authenticate(req.body.username, req.body.password);
  if (user) {
    req.session.userId = user.id;
    req.session.role = user.role;
    res.json({ success: true });
  } else {
    res.status(401).json({ error: 'Invalid credentials' });
  }
});

// Protected route
app.get('/profile', (req, res) => {
  if (!req.session.userId) {
    return res.status(401).json({ error: 'Not authenticated' });
  }
  res.json({ userId: req.session.userId });
});

// Logout
app.post('/logout', (req, res) => {
  req.session.destroy((err) => {
    res.clearCookie('connect.sid');
    res.json({ success: true });
  });
});
```

## Setting Up with Python Flask

```bash
pip install flask flask-session redis
```

```python
from flask import Flask, session, request, jsonify
from flask_session import Session
import redis

app = Flask(__name__)

app.config['SECRET_KEY'] = 'your-secret-key'
app.config['SESSION_TYPE'] = 'redis'
app.config['SESSION_REDIS'] = redis.Redis(host='localhost', port=6379)
app.config['SESSION_PERMANENT'] = False
app.config['SESSION_USE_SIGNER'] = True
app.config['PERMANENT_SESSION_LIFETIME'] = 3600  # 1 hour

Session(app)

@app.route('/login', methods=['POST'])
def login():
    data = request.json
    user = authenticate(data['username'], data['password'])
    if user:
        session['user_id'] = user['id']
        session['username'] = user['username']
        return jsonify({'success': True})
    return jsonify({'error': 'Invalid credentials'}), 401

@app.route('/logout', methods=['POST'])
def logout():
    session.clear()
    return jsonify({'success': True})
```

## What Gets Stored in Redis

Sessions are stored as string keys with a TTL:

```bash
# List all session keys
redis-cli KEYS "sess:*"

# Inspect a session
redis-cli GET "sess:abc123"
# '{"userId":42,"role":"admin","loginTime":1711900000}'

# Check TTL
redis-cli TTL "sess:abc123"
# 86147 (seconds remaining)
```

## Managing Session Expiry

Extend session on each request (sliding expiration):

```javascript
app.use((req, res, next) => {
  if (req.session.userId) {
    // Touch the session to reset TTL on activity
    req.session.touch();
  }
  next();
});
```

Force expire all sessions for a user (useful when changing passwords):

```python
def invalidate_user_sessions(user_id):
    r = redis.Redis()
    cursor = 0
    while True:
        cursor, keys = r.scan(cursor, match='sess:*', count=100)
        for key in keys:
            data = r.get(key)
            if data:
                session_data = json.loads(data)
                if session_data.get('user_id') == user_id:
                    r.delete(key)
        if cursor == 0:
            break
```

## Security Best Practices

```bash
# Always use a strong, random session secret
openssl rand -base64 32

# In redis.conf, bind to localhost only
bind 127.0.0.1

# Require authentication
requirepass your-strong-password
```

In your app configuration:

```javascript
cookie: {
  secure: true,       // HTTPS only
  httpOnly: true,     // Prevent JavaScript access
  sameSite: 'strict', // CSRF protection
  maxAge: 86400000    // 24 hours in milliseconds
}
```

## Summary

Redis provides a shared, fast session store that enables stateless, horizontally scalable web servers. Sessions are stored as JSON strings with TTLs, giving automatic expiry without cleanup jobs. Combined with secure cookie settings, Redis session storage is a reliable foundation for authentication in any multi-server web application.
