# How to Get the Client IPv4 Address from HTTP Requests in Node.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Node.js, Express, IPv4, Networking, HTTP, REST API

Description: Learn how to reliably extract the real client IPv4 address from HTTP requests in Node.js and Express, handling direct connections, reverse proxies, and X-Forwarded-For headers correctly.

## Express: Direct Connection

```javascript
const express = require("express");
const app = express();

app.get("/whoami", (req, res) => {
    // req.socket.remoteAddress - works for direct connections
    // May return "::ffff:192.168.1.1" for IPv4-mapped IPv6
    const raw = req.socket.remoteAddress;
    const ip  = raw.startsWith("::ffff:") ? raw.slice(7) : raw;
    res.json({ client_ip: ip });
});

app.listen(3000);
```

## Express: Behind a Reverse Proxy

```javascript
const express = require("express");
const app = express();

// Trust one level of proxy (Nginx, Traefik, AWS ALB, etc.)
// req.ip will then be the real client IP from X-Forwarded-For
app.set("trust proxy", 1);

app.get("/whoami", (req, res) => {
    res.json({ client_ip: req.ip });
});

app.listen(3000, "127.0.0.1");  // Bind to localhost - Nginx connects here
```

## Express: Manual Header Parsing with Trust Check

```javascript
const net = require("net");

const TRUSTED_PROXY_CIDRS = ["127.0.0.0/8", "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"];

function isTrustedProxy(ip) {
    // Use a CIDR library (e.g. "ipaddr.js") in production
    return ip === "127.0.0.1" || ip.startsWith("10.") || ip.startsWith("192.168.");
}

function getClientIP(req) {
    const remoteAddr = req.socket.remoteAddress.replace("::ffff:", "");
    const xff = req.headers["x-forwarded-for"];
    if (xff && isTrustedProxy(remoteAddr)) {
        return xff.split(",")[0].trim();
    }
    return remoteAddr;
}

const express = require("express");
const app = express();
app.get("/whoami", (req, res) => {
    res.json({ client_ip: getClientIP(req) });
});
app.listen(3000);
```

## raw http Module

```javascript
const http = require("http");

const server = http.createServer((req, res) => {
    const raw = req.socket.remoteAddress || "";
    const ip  = raw.replace("::ffff:", "");
    const xff = req.headers["x-forwarded-for"];
    const clientIP = xff ? xff.split(",")[0].trim() : ip;

    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ client_ip: clientIP }));
});

server.listen(3000, "0.0.0.0");
```

## Logging Middleware

```javascript
const express = require("express");
const app = express();
app.set("trust proxy", 1);

// IP logging middleware
app.use((req, res, next) => {
    const start = Date.now();
    res.on("finish", () => {
        const ms = Date.now() - start;
        console.log(`${req.ip} ${req.method} ${req.path} ${res.statusCode} ${ms}ms`);
    });
    next();
});

app.get("/", (req, res) => res.send("ok"));
app.listen(3000);
```

## Conclusion

`req.socket.remoteAddress` returns the direct connection's IP, which may be `::ffff:x.x.x.x` for IPv4-mapped IPv6 - strip the prefix with `.replace("::ffff:", "")`. Set `app.set("trust proxy", 1)` in Express to safely unwrap `X-Forwarded-For` into `req.ip` when behind one proxy hop. Never trust `X-Forwarded-For` from untrusted sources - always verify that `remoteAddress` is a known proxy before reading forwarded headers.
