# How to Create IPv6 HTTP Servers in Node.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Node.js, IPv6, HTTP, Networking, Dual-Stack

Description: Create IPv6 HTTP servers in Node.js using the built-in http module, extract client IPv6 addresses, and handle dual-stack connections.

## Basic IPv6 HTTP Server

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
    const clientIP = req.socket.remoteAddress;
    const method = req.method;
    const url = req.url;

    console.log(`[${clientIP}] ${method} ${url}`);

    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end(`Hello from IPv6 Node.js! Your IP: ${clientIP}\n`);
});

// '::' listens on all IPv6 interfaces (dual-stack on Linux)
server.listen(8080, '::', () => {
    const addr = server.address();
    console.log(`Server on [${addr.address}]:${addr.port}`);
});
```

## Extracting Real Client IPv6 Address

When behind a proxy, use `X-Forwarded-For`. Handle IPv4-mapped addresses (`::ffff:x.x.x.x`) from dual-stack:

```javascript
const http = require('http');

function getClientIP(req) {
    // Check proxy headers first
    const forwarded = req.headers['x-forwarded-for'];
    if (forwarded) {
        return forwarded.split(',')[0].trim();
    }

    const realIP = req.headers['x-real-ip'];
    if (realIP) return realIP;

    let addr = req.socket.remoteAddress || '';

    // Unwrap IPv4-mapped IPv6: ::ffff:192.168.1.1 → 192.168.1.1
    if (addr.startsWith('::ffff:')) {
        addr = addr.slice(7);
    }

    return addr;
}

const server = http.createServer((req, res) => {
    const ip = getClientIP(req);
    const isIPv6 = ip.includes(':');

    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
        ip,
        version: isIPv6 ? 'IPv6' : 'IPv4',
        path: req.url,
    }));
});

server.listen(8080, '::', () => {
    console.log('Listening on [::]:8080');
});
```

## HTTPS over IPv6

```javascript
const https = require('https');
const fs = require('fs');

const options = {
    key: fs.readFileSync('key.pem'),
    cert: fs.readFileSync('cert.pem'),
};

const server = https.createServer(options, (req, res) => {
    res.writeHead(200);
    res.end('HTTPS over IPv6!\n');
});

server.listen(443, '::', () => {
    console.log('HTTPS server on [::]:443');
});
```

## IPv6-Specific vs Dual-Stack

```javascript
const http = require('http');
const net = require('net');

function createServer(bindAddr, port) {
    const server = http.createServer((req, res) => {
        res.end(`Bound to ${bindAddr}\n`);
    });

    server.listen(port, bindAddr, () => {
        console.log(`Server on [${bindAddr}]:${port}`);
        console.log(`IPv6 only: ${net.isIPv6(bindAddr)}`);
    });

    return server;
}

// IPv6-only (loopback)
createServer('::1', 8081);

// All IPv6 interfaces (+ IPv4 on dual-stack)
createServer('::', 8082);
```

## Middleware for IPv6 Rate Limiting

```javascript
const http = require('http');

const rateLimits = new Map();
const LIMIT = 100;  // requests per minute
const WINDOW_MS = 60_000;

function getRateKey(ip) {
    // Group IPv6 addresses by /64 prefix for rate limiting
    if (ip.includes(':')) {
        const parts = ip.split(':');
        return parts.slice(0, 4).join(':') + '::/64';
    }
    return ip;
}

function checkRateLimit(ip) {
    const key = getRateKey(ip);
    const now = Date.now();
    const record = rateLimits.get(key) || { count: 0, reset: now + WINDOW_MS };

    if (now > record.reset) {
        record.count = 0;
        record.reset = now + WINDOW_MS;
    }

    record.count++;
    rateLimits.set(key, record);

    return record.count <= LIMIT;
}

const server = http.createServer((req, res) => {
    const ip = req.socket.remoteAddress || '';
    const realIP = ip.startsWith('::ffff:') ? ip.slice(7) : ip;

    if (!checkRateLimit(realIP)) {
        res.writeHead(429, { 'Retry-After': '60' });
        res.end('Rate limit exceeded\n');
        return;
    }

    res.writeHead(200);
    res.end('OK\n');
});

server.listen(8080, '::', () => console.log('Rate-limited server on [::]:8080'));
```

## Conclusion

Node.js HTTP servers support IPv6 by passing `'::'` as the hostname to `listen()`. The `req.socket.remoteAddress` property contains the client's IPv6 address, with IPv4 connections appearing as `::ffff:x.x.x.x` on dual-stack servers. Strip the `::ffff:` prefix to get the real IPv4 address. For production deployments behind Nginx or a load balancer, trust `X-Forwarded-For` headers instead. Rate limit IPv6 clients by `/64` prefix since each user may have many individual addresses within their prefix.
