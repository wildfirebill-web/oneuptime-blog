# How to Build a Serverless Session Store with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Serverless, Session

Description: Learn how to implement a serverless session store with Redis to maintain user state across stateless function invocations without relying on sticky routing.

---

Serverless functions are stateless by design. When users log in and navigate across pages, each request may hit a different function instance. Storing session data in Redis gives all instances access to the same user state, enabling stateful experiences on a stateless platform.

## Session Data Structure

A session entry in Redis should include:
- User identity
- Expiration metadata
- Application-specific state (permissions, preferences)

Use Redis hashes for structured session data:

```bash
HSET session:abc123 userId user_42 role admin createdAt 1706054400
EXPIRE session:abc123 3600
```

## Generating Secure Session IDs

```javascript
const crypto = require('crypto');

function generateSessionId() {
  return crypto.randomBytes(32).toString('hex');
}
```

## Session Store Implementation

```javascript
const { createClient } = require('redis');

const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();

const SESSION_TTL = 3600; // 1 hour

async function createSession(userId, userData = {}) {
  const sessionId = generateSessionId();
  const key = `session:${sessionId}`;

  await redis.hSet(key, {
    userId,
    createdAt: Date.now(),
    ...userData
  });
  await redis.expire(key, SESSION_TTL);

  return sessionId;
}

async function getSession(sessionId) {
  const key = `session:${sessionId}`;
  const session = await redis.hGetAll(key);

  if (!session || !session.userId) return null;

  // Refresh TTL on active sessions
  await redis.expire(key, SESSION_TTL);
  return session;
}

async function updateSession(sessionId, updates) {
  const key = `session:${sessionId}`;
  await redis.hSet(key, updates);
  await redis.expire(key, SESSION_TTL);
}

async function destroySession(sessionId) {
  await redis.del(`session:${sessionId}`);
}
```

## Lambda Handler with Session Management

```javascript
exports.handler = async (event) => {
  const sessionId = event.headers?.cookie?.match(/sessionId=([^;]+)/)?.[1];

  if (!sessionId) {
    return {
      statusCode: 401,
      body: JSON.stringify({ error: 'No session' })
    };
  }

  const session = await getSession(sessionId);
  if (!session) {
    return {
      statusCode: 401,
      body: JSON.stringify({ error: 'Session expired or invalid' })
    };
  }

  // Attach session to request context
  const response = await handleRequest(event, session);
  return response;
};
```

## Login Handler

```javascript
exports.login = async (event) => {
  const { username, password } = JSON.parse(event.body);

  const user = await authenticateUser(username, password);
  if (!user) {
    return { statusCode: 401, body: JSON.stringify({ error: 'Invalid credentials' }) };
  }

  const sessionId = await createSession(user.id, {
    email: user.email,
    role: user.role
  });

  return {
    statusCode: 200,
    headers: {
      'Set-Cookie': `sessionId=${sessionId}; HttpOnly; Secure; SameSite=Strict; Max-Age=3600`
    },
    body: JSON.stringify({ ok: true })
  };
};
```

## Session Listing for Admin Use

```bash
redis-cli keys "session:*" | wc -l
redis-cli hgetall "session:abc123"
```

## Summary

Redis hash-based session storage works well in serverless environments because all function instances share the same store. The key patterns are generating cryptographically random session IDs, storing structured data in hashes, refreshing TTLs on active sessions to extend their lifetime, and cleaning up sessions on logout. This approach is portable across AWS Lambda, Azure Functions, and Google Cloud Functions.
