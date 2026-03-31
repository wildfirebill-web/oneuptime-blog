# How to Build a Session Store in Python with Redis

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Python, Session Management, Flask, Authentication

Description: Learn how to build a Redis-backed session store in Python, including session creation, storage, expiry, and Flask integration with flask-session.

---

## Why Use Redis for Sessions?

Redis is an excellent session store because:
- In-memory speed for fast session lookups on every request
- TTL-based automatic expiry matches session timeout requirements
- Works across multiple application servers (stateless apps)
- Easy to invalidate individual or all sessions

## Basic Session Manager

```python
import redis
import uuid
import json
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

class SessionStore:
    def __init__(self, prefix='session:', ttl=3600):
        self.prefix = prefix
        self.ttl = ttl

    def _key(self, session_id):
        return f"{self.prefix}{session_id}"

    def create(self, user_data: dict) -> str:
        """Create a new session and return the session ID."""
        session_id = str(uuid.uuid4())
        key = self._key(session_id)

        session_data = {
            **user_data,
            'created_at': int(time.time()),
            'last_accessed': int(time.time())
        }

        r.setex(key, self.ttl, json.dumps(session_data))
        return session_id

    def get(self, session_id: str) -> dict | None:
        """Retrieve session data, refreshing TTL on access."""
        key = self._key(session_id)
        raw = r.get(key)

        if not raw:
            return None

        session_data = json.loads(raw)
        session_data['last_accessed'] = int(time.time())

        # Refresh TTL on access (sliding expiry)
        r.setex(key, self.ttl, json.dumps(session_data))
        return session_data

    def update(self, session_id: str, updates: dict) -> bool:
        """Update specific fields in a session."""
        data = self.get(session_id)
        if not data:
            return False

        data.update(updates)
        key = self._key(session_id)
        r.setex(key, self.ttl, json.dumps(data))
        return True

    def delete(self, session_id: str) -> bool:
        """Delete a session (logout)."""
        key = self._key(session_id)
        return bool(r.delete(key))

    def exists(self, session_id: str) -> bool:
        """Check if a session exists."""
        return bool(r.exists(self._key(session_id)))

    def ttl_remaining(self, session_id: str) -> int:
        """Get remaining TTL in seconds."""
        return r.ttl(self._key(session_id))

# Usage
store = SessionStore(ttl=3600)

# Create session on login
session_id = store.create({'user_id': 1001, 'email': 'alice@example.com', 'role': 'user'})
print(f"Session created: {session_id}")

# Retrieve session on request
session = store.get(session_id)
print(f"User: {session['email']}")

# Update session
store.update(session_id, {'last_page': '/dashboard'})

# Delete session on logout
store.delete(session_id)
```

## Using Redis Hashes for Sessions

```python
import redis
import uuid
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

class HashSessionStore:
    """Uses Redis Hashes for individual field access."""

    def __init__(self, prefix='session:', ttl=3600):
        self.prefix = prefix
        self.ttl = ttl

    def _key(self, session_id):
        return f"{self.prefix}{session_id}"

    def create(self, user_id: str, **fields) -> str:
        session_id = str(uuid.uuid4())
        key = self._key(session_id)

        r.hset(key, mapping={
            'user_id': user_id,
            'created_at': int(time.time()),
            **fields
        })
        r.expire(key, self.ttl)
        return session_id

    def get_field(self, session_id: str, field: str) -> str | None:
        return r.hget(self._key(session_id), field)

    def get_all(self, session_id: str) -> dict:
        return r.hgetall(self._key(session_id))

    def set_field(self, session_id: str, field: str, value: str):
        key = self._key(session_id)
        r.hset(key, field, value)
        r.expire(key, self.ttl)

    def delete(self, session_id: str):
        r.delete(self._key(session_id))

store = HashSessionStore()
sid = store.create('1001', email='alice@example.com', role='admin')
print(store.get_field(sid, 'email'))
print(store.get_all(sid))
```

## Flask Integration with flask-session

```bash
pip install flask flask-session
```

```python
from flask import Flask, session, request, jsonify, redirect, url_for
from flask_session import Session
import redis

app = Flask(__name__)

# Flask session configuration
app.config['SECRET_KEY'] = 'your-very-secret-key-here'
app.config['SESSION_TYPE'] = 'redis'
app.config['SESSION_REDIS'] = redis.from_url('redis://localhost:6379')
app.config['SESSION_PERMANENT'] = False
app.config['PERMANENT_SESSION_LIFETIME'] = 3600  # 1 hour

Session(app)

@app.route('/login', methods=['POST'])
def login():
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')

    # Validate credentials (simplified)
    if username == 'admin' and password == 'secret':
        session['user_id'] = 1001
        session['username'] = username
        session['role'] = 'admin'
        return jsonify({'message': 'Login successful'})

    return jsonify({'error': 'Invalid credentials'}), 401

@app.route('/profile')
def profile():
    if 'user_id' not in session:
        return jsonify({'error': 'Not authenticated'}), 401

    return jsonify({
        'user_id': session['user_id'],
        'username': session['username'],
        'role': session['role']
    })

@app.route('/logout', methods=['POST'])
def logout():
    session.clear()
    return jsonify({'message': 'Logged out'})
```

## Tracking Active Sessions per User

```python
import redis
import uuid
import json
import time

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

def create_session_with_tracking(user_id: str, device_info: str) -> str:
    session_id = str(uuid.uuid4())

    # Store session data
    session_key = f"session:{session_id}"
    r.setex(session_key, 3600, json.dumps({
        'user_id': user_id,
        'device': device_info,
        'created_at': int(time.time())
    }))

    # Track user's sessions in a sorted set (score = creation time)
    user_sessions_key = f"user:sessions:{user_id}"
    r.zadd(user_sessions_key, {session_id: time.time()})
    r.expire(user_sessions_key, 3600)

    return session_id

def get_user_sessions(user_id: str) -> list:
    """Get all active sessions for a user."""
    user_sessions_key = f"user:sessions:{user_id}"
    session_ids = r.zrange(user_sessions_key, 0, -1)

    sessions = []
    for sid in session_ids:
        data = r.get(f"session:{sid}")
        if data:
            sessions.append({'id': sid, **json.loads(data)})
        else:
            # Clean up expired session from tracking set
            r.zrem(user_sessions_key, sid)

    return sessions

def logout_all_devices(user_id: str):
    """Invalidate all sessions for a user."""
    user_sessions_key = f"user:sessions:{user_id}"
    session_ids = r.zrange(user_sessions_key, 0, -1)

    pipe = r.pipeline()
    for sid in session_ids:
        pipe.delete(f"session:{sid}")
    pipe.delete(user_sessions_key)
    pipe.execute()
    print(f"Logged out {len(session_ids)} sessions")
```

## Summary

Redis session stores in Python combine UUID-based session IDs with JSON or Hash storage for flexible session management. Use `SETEX` for automatic TTL-based expiry, sliding expiry by refreshing TTL on each access, and Redis Sorted Sets for tracking multiple sessions per user. The `flask-session` library provides seamless Redis session integration for Flask applications with minimal configuration.
