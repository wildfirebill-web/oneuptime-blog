# How to Use Redis as a Session Store for Your Web App

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Sessions, Authentication, Web, Beginner, Node.js, Express

Description: A beginner-friendly guide to storing web application sessions in Redis, replacing cookie-based storage for scalable, server-side session management.

---

## Why Use Redis for Sessions?

By default, many web frameworks store sessions in memory on the server or in cookies. Memory storage doesn't work when you have multiple servers. Cookie storage has size limits and security concerns. Redis solves both problems:

- Sessions survive server restarts
- Multiple servers can read the same session
- Sessions expire automatically with TTL
- Scales to millions of sessions

## Setting Up Express with Redis Sessions

```bash
npm install express express-session connect-redis ioredis
```

```javascript
const express = require('express');
const session = require('express-session');
const { RedisStore } = require('connect-redis');
const Redis = require('ioredis');

const app = express();

// Create Redis client
const redisClient = new Redis({ host: 'localhost', port: 6379 });

// Configure session middleware
app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET || 'your-secret-key',
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production', // HTTPS only in production
    httpOnly: true,   // Prevent JavaScript access
    maxAge: 86400000  // 24 hours in milliseconds
  }
}));

// Login endpoint
app.post('/login', async (req, res) => {
  const { username, password } = req.body;

  const user = await authenticateUser(username, password);
  if (!user) return res.status(401).json({ error: 'Invalid credentials' });

  // Store user info in session
  req.session.userId = user.id;
  req.session.username = user.username;
  req.session.roles = user.roles;

  res.json({ message: 'Logged in', userId: user.id });
});

// Protected route
app.get('/profile', requireAuth, (req, res) => {
  res.json({ userId: req.session.userId, username: req.session.username });
});

// Logout endpoint
app.post('/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) return res.status(500).json({ error: 'Logout failed' });
    res.clearCookie('connect.sid');
    res.json({ message: 'Logged out' });
  });
});

// Auth middleware
function requireAuth(req, res, next) {
  if (!req.session.userId) {
    return res.status(401).json({ error: 'Not authenticated' });
  }
  next();
}

app.listen(3000, () => console.log('Server running on port 3000'));
```

## What Gets Stored in Redis

When a user logs in, connect-redis automatically stores the session as:

```bash
# Key format: sess:{sessionId}
GET sess:abc123xyz456

# Example value:
# {"cookie":{"originalMaxAge":86400000,"expires":"2026-04-01T00:00:00.000Z","httpOnly":true},"userId":42,"username":"alice","roles":["user"]}

# Check TTL
TTL sess:abc123xyz456
# 86387 (seconds remaining)
```

## Manual Session Management (Without connect-redis)

```javascript
const crypto = require('crypto');

function generateSessionId() {
  return crypto.randomBytes(32).toString('hex');
}

async function createSession(userId, userData) {
  const sessionId = generateSessionId();
  const sessionData = { userId, ...userData, createdAt: Date.now() };

  await redisClient.setex(
    `session:${sessionId}`,
    86400, // 24 hours
    JSON.stringify(sessionData)
  );

  return sessionId;
}

async function getSession(sessionId) {
  const data = await redisClient.get(`session:${sessionId}`);
  return data ? JSON.parse(data) : null;
}

async function destroySession(sessionId) {
  await redisClient.del(`session:${sessionId}`);
}

// Manual session middleware
async function sessionMiddleware(req, res, next) {
  const sessionId = req.cookies?.sessionId;
  if (sessionId) {
    req.session = await getSession(sessionId);
  }
  next();
}
```

## Python Flask with Redis Sessions

```bash
pip install flask flask-session redis
```

```python
from flask import Flask, session, request, jsonify
from flask_session import Session
import redis

app = Flask(__name__)

# Configure Flask-Session to use Redis
app.config['SECRET_KEY'] = 'your-secret-key'
app.config['SESSION_TYPE'] = 'redis'
app.config['SESSION_REDIS'] = redis.Redis(host='localhost', port=6379)
app.config['SESSION_PERMANENT'] = False
app.config['SESSION_USE_SIGNER'] = True
app.config['SESSION_KEY_PREFIX'] = 'session:'

Session(app)

@app.route('/login', methods=['POST'])
def login():
    data = request.json
    user = authenticate(data['username'], data['password'])
    if not user:
        return jsonify({'error': 'Invalid credentials'}), 401

    session['user_id'] = user['id']
    session['username'] = user['username']
    return jsonify({'message': 'Logged in'})

@app.route('/profile')
def profile():
    if 'user_id' not in session:
        return jsonify({'error': 'Not authenticated'}), 401
    return jsonify({'userId': session['user_id']})

@app.route('/logout', methods=['POST'])
def logout():
    session.clear()
    return jsonify({'message': 'Logged out'})
```

## Session Security Best Practices

```javascript
// 1. Use a strong secret key
const SECRET = process.env.SESSION_SECRET; // Never hardcode

// 2. Regenerate session ID after login (prevents session fixation)
app.post('/login', async (req, res) => {
  const user = await authenticate(req.body);
  if (!user) return res.status(401).json({ error: 'Invalid' });

  // Regenerate prevents session fixation attacks
  req.session.regenerate((err) => {
    req.session.userId = user.id;
    res.json({ message: 'Logged in' });
  });
});

// 3. Set secure cookie options
const sessionOptions = {
  cookie: {
    secure: true,    // HTTPS only
    httpOnly: true,  // No JavaScript access
    sameSite: 'strict', // CSRF protection
    maxAge: 86400000
  }
};
```

## Viewing Active Sessions in Redis

```bash
# Count active sessions
redis-cli KEYS "sess:*" | wc -l

# View a session (replace with actual session ID)
redis-cli GET "sess:your-session-id-here"

# Delete a specific session (force logout)
redis-cli DEL "sess:your-session-id-here"
```

## Summary

Redis is the standard choice for web session storage in multi-server deployments. Libraries like connect-redis (Node.js) and Flask-Session (Python) handle the integration automatically. Sessions are stored as JSON with automatic TTL expiration. Always use secure cookies, regenerate session IDs after login, and store session secrets in environment variables rather than code.
