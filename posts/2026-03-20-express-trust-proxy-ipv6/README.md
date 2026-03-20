# How to Configure Express.js Trust Proxy for IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Express.js, Node.js, IPv6, Trust Proxy, X-Forwarded-For, Security, Middleware

Description: Configure Express.js trust proxy settings to correctly extract real client IPv6 addresses from X-Forwarded-For headers when behind IPv6-capable load balancers and reverse proxies.

## Introduction

When Express.js runs behind a reverse proxy, the client IP in `req.ip` is the proxy's IP unless trust proxy is configured. For IPv6 environments, this means correctly trusting IPv6 proxy addresses and parsing IPv6 addresses from `X-Forwarded-For` headers.

## How X-Forwarded-For Works with IPv6

```
Client 2001:db8::cafe:1
    → IPv6 Load Balancer (2001:db8::lb)
    → NGINX Proxy (::1)
    → Express.js

X-Forwarded-For: 2001:db8::cafe:1, 2001:db8::lb
X-Real-IP: 2001:db8::cafe:1
```

## Step 1: Trust Proxy Settings

```javascript
const express = require('express');
const app = express();

// Option 1: Trust 1 hop (most common — direct NGINX proxy)
app.set('trust proxy', 1);
// req.ip = first untrusted address in X-Forwarded-For

// Option 2: Trust a specific IPv6 address
app.set('trust proxy', '::1');
// Express trusts ::1 as a proxy

// Option 3: Trust multiple IPv6 addresses/subnets
app.set('trust proxy', ['::1', '2001:db8::/32', 'loopback']);

// Option 4: Trust a count of hops
app.set('trust proxy', 2);  // Trust 2 proxy hops

// Option 5: Trust all (dangerous — do not use with public internet)
// app.set('trust proxy', true);
```

## Step 2: Extract Real IPv6 Address

```javascript
// middleware/realIP.js
const { isIPv4, isIPv6 } = require('net');

function normalizeIPv6(ip) {
    if (!ip) return null;

    // Remove IPv6 brackets [::1] → ::1
    ip = ip.replace(/^\[/, '').replace(/\]$/, '');

    // Remove IPv4-mapped IPv6 prefix ::ffff:1.2.3.4 → 1.2.3.4
    if (ip.startsWith('::ffff:')) {
        const v4 = ip.slice(7);
        if (isIPv4(v4)) return v4;
    }

    return ip;
}

function realIPMiddleware(req, res, next) {
    // With trust proxy set, req.ip is already the real IP
    // But we also handle the case where it's not set
    let ip = req.ip;

    if (!ip) {
        const xff = req.headers['x-forwarded-for'];
        if (xff) {
            ip = xff.split(',')[0].trim();
        } else {
            ip = req.connection.remoteAddress;
        }
    }

    req.realIP = normalizeIPv6(ip);
    req.isIPv6 = isIPv6(req.realIP);
    next();
}

module.exports = realIPMiddleware;
```

## Step 3: Validate Trust Proxy Configuration

```javascript
// test-trust-proxy.js
const express = require('express');
const app = express();

// Configure trust proxy
app.set('trust proxy', ['::1', 'loopback']);

app.get('/debug-ip', (req, res) => {
    res.json({
        // With trust proxy, req.ip is the real client IP
        'req.ip':                  req.ip,
        'req.ips':                 req.ips,  // Full chain
        'x-forwarded-for':         req.headers['x-forwarded-for'],
        'connection.remoteAddress': req.socket.remoteAddress,
    });
});

app.listen('[::]:3000', '::');
```

```bash
# Test — simulate a request from IPv6 client via proxy
curl -6 http://[::1]:3000/debug-ip \
    -H "X-Forwarded-For: 2001:db8::cafe"
# Expected: req.ip = "2001:db8::cafe"

# Without the header (proxy IP shows)
curl -6 http://[::1]:3000/debug-ip
# Expected: req.ip = "::1"
```

## Step 4: Security Considerations

```javascript
// Security: never trust X-Forwarded-For from untrusted sources

// BAD: Always trusts XFF header, allows IP spoofing
app.set('trust proxy', true);

// GOOD: Trust only your known IPv6 proxy
app.set('trust proxy', '2001:db8::proxy');

// GOOD: Trust only loopback (NGINX on same host)
app.set('trust proxy', 'loopback');

// Rate limiting using real IP (must be after trust proxy config)
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
    windowMs: 60 * 1000,
    max: 100,
    // req.ip is correct after trust proxy
    keyGenerator: (req) => {
        // For IPv6, rate limit by /64 subnet
        const ip = req.ip;
        if (require('net').isIPv6(ip)) {
            const parts = ip.split(':');
            return parts.slice(0, 4).join(':') + '::/64';
        }
        return ip;
    },
});
app.use('/api/', limiter);
```

## Step 5: NGINX Configuration to Forward IPv6

```nginx
server {
    listen [::]:80;

    location / {
        proxy_pass http://[::1]:3000;

        # Set X-Forwarded-For to real IPv6 client address
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
    }
}
```

## Conclusion

Express.js trust proxy for IPv6 requires listing your IPv6 proxy addresses: `app.set('trust proxy', ['::1', '2001:db8::/32'])`. Once configured, `req.ip` returns the real IPv6 client address from `X-Forwarded-For`. Never use `trust proxy: true` on public servers — restrict to known proxy addresses to prevent IP spoofing. Monitor Express.js with OneUptime to verify correct IP extraction in logs.
