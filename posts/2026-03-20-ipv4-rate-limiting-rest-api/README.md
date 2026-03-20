# How to Implement IPv4 Address-Based Rate Limiting in REST APIs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: REST API, IPv4, Rate Limiting, Python, Node.js, Redis

Description: Learn how to implement per-IP rate limiting in REST APIs using sliding window and token bucket algorithms in Python and Node.js, with Redis for distributed deployments.

## Python / Flask: Simple In-Memory Rate Limiter

```python
from flask import Flask, request, jsonify
from collections import defaultdict
import time, threading

app   = Flask(__name__)
_lock = threading.Lock()

# Sliding window: {ip: [timestamp, ...]}

_windows: dict[str, list[float]] = defaultdict(list)

LIMIT   = 100    # requests
WINDOW  = 60.0   # seconds

def is_rate_limited(ip: str) -> bool:
    now = time.monotonic()
    with _lock:
        ts = _windows[ip]
        # Remove timestamps outside the window
        cutoff = now - WINDOW
        _windows[ip] = [t for t in ts if t > cutoff]
        if len(_windows[ip]) >= LIMIT:
            return True
        _windows[ip].append(now)
        return False

@app.before_request
def rate_limit():
    ip = request.headers.get("X-Forwarded-For", request.remote_addr)
    ip = ip.split(",")[0].strip()
    if is_rate_limited(ip):
        return jsonify(error="rate limit exceeded"), 429

@app.get("/api/data")
def data():
    return jsonify(result="ok")
```

## Python / Flask + Redis: Distributed Rate Limiter

```python
import redis
from flask import Flask, request, jsonify

app = Flask(__name__)
r   = redis.Redis(host="localhost", port=6379, decode_responses=True)

LIMIT  = 100
WINDOW = 60  # seconds

def check_rate_limit(ip: str) -> tuple[bool, int]:
    """Returns (is_limited, remaining)."""
    key = f"rl:{ip}"
    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, WINDOW)
    count, _ = pipe.execute()
    remaining = max(0, LIMIT - count)
    return count > LIMIT, remaining

@app.before_request
def rate_limit():
    ip = request.headers.get("X-Forwarded-For", request.remote_addr)
    ip = ip.split(",")[0].strip()
    limited, remaining = check_rate_limit(ip)
    if limited:
        resp = jsonify(error="Too Many Requests")
        resp.headers["X-RateLimit-Limit"]     = LIMIT
        resp.headers["X-RateLimit-Remaining"] = 0
        resp.headers["Retry-After"]           = WINDOW
        return resp, 429
```

## Node.js / Express: express-rate-limit

```javascript
const express     = require("express");
const rateLimit   = require("express-rate-limit");
const RedisStore  = require("rate-limit-redis");
const redis       = require("redis");

const app = express();
app.set("trust proxy", 1);  // Required to get real IP behind proxy

const client = redis.createClient({ url: "redis://localhost:6379" });
client.connect();

const limiter = rateLimit({
    windowMs: 60 * 1000,   // 1 minute
    max: 100,              // requests per window per IP
    standardHeaders: true, // Return X-RateLimit-* headers
    legacyHeaders: false,
    store: new RedisStore({ sendCommand: (...args) => client.sendCommand(args) }),
    keyGenerator: (req) => req.ip,  // req.ip respects trust proxy setting
    handler: (req, res) => {
        res.status(429).json({ error: "Too Many Requests" });
    },
});

app.use("/api/", limiter);

app.get("/api/data", (req, res) => {
    res.json({ result: "ok" });
});

app.listen(3000);
```

## Rate Limiting Headers

| Header | Meaning |
|--------|---------|
| `X-RateLimit-Limit` | Max requests per window |
| `X-RateLimit-Remaining` | Remaining requests in current window |
| `X-RateLimit-Reset` | Unix timestamp when window resets |
| `Retry-After` | Seconds until client can retry (on 429) |

## Conclusion

In-memory rate limiting works for single-instance deployments but loses state on restart and doesn't share across replicas. Redis-backed rate limiting is the production standard for distributed systems. Always extract the real client IP before using it as the rate limit key - check `X-Forwarded-For` only when `remote_addr` is a trusted proxy. Return standard `X-RateLimit-*` headers and `Retry-After` on 429 responses to help well-behaved clients back off gracefully.
