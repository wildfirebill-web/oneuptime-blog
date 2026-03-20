# How to Use IPv6 with Express.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Node.js, IPv6, Express.js, HTTP, REST API, Middleware

Description: Build IPv6-capable Express.js applications including server binding, client IP extraction middleware, and IPv6-aware routing.

## Binding Express to IPv6

```javascript
const express = require('express');

const app = express();

app.get('/', (req, res) => {
    res.send('Hello from IPv6 Express!');
});

// Listen on all IPv6 interfaces
// Second argument is the host — '::' enables dual-stack
app.listen(3000, '::', () => {
    console.log('Express server on [::]:3000');
});
```

## Client IP Extraction Middleware

Express populates `req.ip` when `trust proxy` is configured:

```javascript
const express = require('express');

const app = express();

// Trust first proxy (for deployments behind Nginx/load balancer)
app.set('trust proxy', 1);

// Middleware to normalize and log client IP
app.use((req, res, next) => {
    let ip = req.ip || req.socket.remoteAddress || '';

    // Unwrap IPv4-mapped addresses from dual-stack
    if (ip.startsWith('::ffff:')) {
        ip = ip.slice(7);
    }

    req.clientIP = ip;
    req.isIPv6 = ip.includes(':');

    console.log(`[${ip}] ${req.method} ${req.path}`);
    next();
});

app.get('/me', (req, res) => {
    res.json({
        ip: req.clientIP,
        version: req.isIPv6 ? 'IPv6' : 'IPv4',
    });
});

app.listen(3000, '::', () => console.log('Listening on [::]:3000'));
```

## IPv6 Validation Middleware

```javascript
const express = require('express');
const net = require('net');

const app = express();
app.use(express.json());

// Middleware to validate IPv6 address in request body
function validateIPv6Body(field) {
    return (req, res, next) => {
        const value = req.body[field];
        if (!value) {
            return res.status(400).json({ error: `${field} is required` });
        }
        if (!net.isIPv6(value)) {
            return res.status(400).json({
                error: `${field} must be a valid IPv6 address`,
                received: value,
            });
        }
        next();
    };
}

app.post('/device', validateIPv6Body('ipv6Address'), (req, res) => {
    const { name, ipv6Address } = req.body;
    res.json({
        message: 'Device registered',
        name,
        ipv6Address,
    });
});

app.listen(3000, '::', () => console.log('Validation server on [::]:3000'));
```

## Rate Limiting by IPv6 /64 Prefix

```javascript
const express = require('express');

const app = express();
const requests = new Map();

function getIPv6Prefix(ip) {
    // For IPv6, rate-limit by /64 prefix
    if (ip.includes(':')) {
        const parts = ip.split(':');
        return parts.slice(0, 4).join(':') + '::/64';
    }
    return ip;
}

app.use((req, res, next) => {
    let ip = req.socket.remoteAddress || '';
    if (ip.startsWith('::ffff:')) ip = ip.slice(7);

    const key = getIPv6Prefix(ip);
    const now = Date.now();
    const windowMs = 60_000;
    const limit = 60;

    const rec = requests.get(key) || { count: 0, reset: now + windowMs };
    if (now > rec.reset) {
        rec.count = 0;
        rec.reset = now + windowMs;
    }
    rec.count++;
    requests.set(key, rec);

    if (rec.count > limit) {
        return res.status(429).json({
            error: 'Rate limit exceeded',
            retryAfter: Math.ceil((rec.reset - now) / 1000),
        });
    }

    res.setHeader('X-RateLimit-Remaining', Math.max(0, limit - rec.count));
    next();
});

app.get('/', (req, res) => res.json({ status: 'ok' }));
app.listen(3000, '::', () => console.log('Rate-limited Express on [::]:3000'));
```

## IPv6 in Express Routes

```javascript
const express = require('express');
const net = require('net');

const app = express();
app.use(express.json());

// Database of IPv6-addressed resources
const devices = new Map();

// GET /devices/:ipv6 — URL-encode brackets or use hex without brackets
app.get('/devices/:address', (req, res) => {
    const address = decodeURIComponent(req.params.address);

    if (!net.isIPv6(address)) {
        return res.status(400).json({ error: 'Invalid IPv6 address' });
    }

    // Normalize to compressed form
    // In production, use a library like 'ip6' for normalization
    const device = devices.get(address);
    if (!device) return res.status(404).json({ error: 'Not found' });
    res.json(device);
});

app.post('/devices', (req, res) => {
    const { address, name } = req.body;

    if (!net.isIPv6(address)) {
        return res.status(400).json({ error: 'Invalid IPv6 address' });
    }

    devices.set(address, { address, name, registered: new Date() });
    res.status(201).json({ message: 'Registered', address });
});

app.listen(3000, '::', () => console.log('Device API on [::]:3000'));
```

## Conclusion

Express.js supports IPv6 by passing `'::'` as the host to `listen()`. Set `app.set('trust proxy', 1)` when behind a proxy to populate `req.ip` from `X-Forwarded-For`. Normalize client IPs by stripping `::ffff:` from IPv4-mapped addresses. Use `net.isIPv6()` for input validation. Rate-limit by `/64` prefix since a single IPv6 user may rotate through many addresses in their prefix. IPv6 literals in URL paths must be URL-encoded since brackets have special meaning in URLs.
