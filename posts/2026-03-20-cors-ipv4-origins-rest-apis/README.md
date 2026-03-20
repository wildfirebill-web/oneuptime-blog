# How to Configure CORS with IPv4 Origins in REST APIs

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: CORS, IPv4, REST API, Node.js, Python, Nginx, Security

Description: Configure Cross-Origin Resource Sharing (CORS) to allow specific IPv4 origins in Node.js Express, Python FastAPI, and Nginx so that browser-based clients on private networks can access REST APIs.

## Introduction

When a web application is hosted at an IPv4 address (e.g., `http://192.168.1.50`) and calls an API at a different IP, the browser enforces CORS. You must explicitly allow these IPv4 origins on the server side.

## Node.js with Express and cors Package

```javascript
const express = require('express');
const cors = require('cors');

const app = express();

const allowedOrigins = [
    'http://192.168.1.50',
    'http://192.168.1.50:3000',
    'http://10.0.0.5:8080',
];

app.use(cors({
    origin: (origin, callback) => {
        if (!origin || allowedOrigins.includes(origin)) {
            callback(null, true);
        } else {
            callback(new Error(`CORS blocked: ${origin}`));
        }
    },
    methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization'],
    credentials: true,
}));

app.get('/api/data', (req, res) => res.json({ status: 'ok' }));
app.listen(3000, '0.0.0.0');
```

## Python FastAPI

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

allowed_origins = [
    "http://192.168.1.50",
    "http://192.168.1.50:3000",
    "http://10.0.0.5:8080",
]

app.add_middleware(
    CORSMiddleware,
    allow_origins=allowed_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/api/data")
def get_data():
    return {"status": "ok"}
```

## Python Flask

```python
from flask import Flask
from flask_cors import CORS

app = Flask(__name__)

CORS(app, origins=[
    "http://192.168.1.50",
    "http://192.168.1.50:3000",
    "http://10.0.0.5",
], supports_credentials=True)

@app.route("/api/data")
def data():
    return {"status": "ok"}
```

## Nginx CORS Headers

```nginx
server {
    listen 8000;

    location /api/ {
        # Map allowed IPv4 origins
        set $cors_origin "";
        if ($http_origin ~* "^http://(192\.168\.1\.50|10\.0\.0\.5)(:\d+)?$") {
            set $cors_origin $http_origin;
        }

        add_header 'Access-Control-Allow-Origin' $cors_origin always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;

        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Max-Age' 86400;
            return 204;
        }

        proxy_pass http://127.0.0.1:3000;
    }
}
```

## Allowing a Subnet (Development Only)

```javascript
// Danger: only use in trusted development networks
const subnet = /^http:\/\/192\.168\.\d{1,3}\.\d{1,3}(:\d+)?$/;

app.use(cors({
    origin: (origin, cb) => cb(null, !origin || subnet.test(origin)),
}));
```

## Conclusion

Always specify explicit IPv4 origins rather than using wildcard `*` when `credentials: true` is required. Match origins using exact string comparison or a carefully constrained regex. For production, maintain an allow-list rather than an allow-subnet pattern.
