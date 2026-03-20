# How to Build a Reverse Proxy in Node.js for IPv4 Traffic

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Node.js, Reverse Proxy, IPv4, HTTP, Http-proxy, Networking

Description: Learn how to build a reverse proxy in Node.js that forwards IPv4 HTTP traffic to upstream servers using the http-proxy library and native http module.

## Using http-proxy Library

```bash
npm install http-proxy
```

```javascript
const httpProxy = require('http-proxy');
const http = require('http');
const net = require('net');

const UPSTREAM = 'http://127.0.0.1:3000';

// Create the proxy
const proxy = httpProxy.createProxyServer({
    target: UPSTREAM,
    // Force IPv4 on the upstream connection
    agent: new http.Agent({ family: 4 }),
    changeOrigin: true,
});

// Handle proxy errors
proxy.on('error', (err, req, res) => {
    console.error(`Proxy error: ${err.message}`);
    if (!res.headersSent) {
        res.writeHead(502, { 'Content-Type': 'text/plain' });
    }
    res.end('Bad Gateway');
});

// Log each proxied request
proxy.on('proxyReq', (proxyReq, req, res, options) => {
    const clientIP = req.socket.remoteAddress;
    proxyReq.setHeader('X-Forwarded-For', clientIP);
    proxyReq.setHeader('X-Real-IP', clientIP);
    console.log(`Proxying: ${req.method} ${req.url} from ${clientIP}`);
});

// Create the proxy server
const server = http.createServer((req, res) => {
    proxy.web(req, res);
});

server.listen(8080, '0.0.0.0', () => {
    console.log(`Reverse proxy on :8080 -> ${UPSTREAM}`);
});
```

## Simple Reverse Proxy Without Dependencies

```javascript
const http = require('http');
const net = require('net');

const UPSTREAM_HOST = '127.0.0.1';
const UPSTREAM_PORT = 3000;

function forwardRequest(clientReq, clientRes) {
    const options = {
        hostname: UPSTREAM_HOST,
        port: UPSTREAM_PORT,
        path: clientReq.url,
        method: clientReq.method,
        headers: {
            ...clientReq.headers,
            'X-Forwarded-For': clientReq.socket.remoteAddress,
            'X-Forwarded-Host': clientReq.headers.host,
        },
        family: 4,  // Force IPv4
    };

    const upstreamReq = http.request(options, (upstreamRes) => {
        // Forward status and headers to client
        clientRes.writeHead(upstreamRes.statusCode, upstreamRes.headers);
        // Pipe the upstream response body to the client
        upstreamRes.pipe(clientRes);
    });

    upstreamReq.on('error', (err) => {
        console.error(`Upstream error: ${err.message}`);
        clientRes.writeHead(502);
        clientRes.end('Bad Gateway');
    });

    // Pipe the client request body to the upstream
    clientReq.pipe(upstreamReq);
}

const server = http.createServer(forwardRequest);

server.listen(8080, '0.0.0.0', () => {
    console.log('Proxy listening on port 8080');
});
```

## Reverse Proxy with URL Routing

```javascript
const http = require('http');

const routes = {
    '/api/': { host: '127.0.0.1', port: 3001 },
    '/auth/': { host: '127.0.0.1', port: 3002 },
    '/': { host: '127.0.0.1', port: 3000 },  // Default
};

function getUpstream(url) {
    for (const [prefix, upstream] of Object.entries(routes)) {
        if (url.startsWith(prefix)) return upstream;
    }
    return routes['/'];
}

const server = http.createServer((clientReq, clientRes) => {
    const upstream = getUpstream(clientReq.url);

    const options = {
        hostname: upstream.host,
        port: upstream.port,
        path: clientReq.url,
        method: clientReq.method,
        headers: clientReq.headers,
        family: 4,
    };

    const upstreamReq = http.request(options, (upstreamRes) => {
        clientRes.writeHead(upstreamRes.statusCode, upstreamRes.headers);
        upstreamRes.pipe(clientRes);
    });

    upstreamReq.on('error', () => { clientRes.writeHead(502); clientRes.end(); });
    clientReq.pipe(upstreamReq);
});

server.listen(8080, '0.0.0.0');
```

## Conclusion

A Node.js reverse proxy can be built with the `http-proxy` library for production use or with raw `http.request()` + piping for minimal dependencies. Always set `X-Forwarded-For` to preserve the real client IP, and pass `family: 4` to the upstream agent to enforce IPv4 connections. For production, consider Nginx or a dedicated proxy service.
