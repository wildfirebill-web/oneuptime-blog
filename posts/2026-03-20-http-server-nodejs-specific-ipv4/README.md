# How to Create an HTTP Server in Node.js Bound to a Specific IPv4 Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Node.js, HTTP, IPv4, Server, Networking, Express

Description: Learn how to bind a Node.js HTTP server to a specific IPv4 address using the http module and Express, controlling which network interface it listens on.

## Native http Module

```javascript
const http = require('http');

const HOST = '192.168.1.50';  // Bind to this specific IPv4 interface
const PORT = 8080;

const server = http.createServer((req, res) => {
    const clientIP = req.socket.remoteAddress;
    console.log(`Request from ${clientIP}: ${req.method} ${req.url}`);

    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
        message: 'Hello!',
        serverAddress: HOST,
        clientIP,
        path: req.url,
    }));
});

// Pass the host to listen() to bind to a specific IP
server.listen(PORT, HOST, () => {
    const addr = server.address();
    console.log(`HTTP server listening on ${addr.address}:${addr.port}`);
});

server.on('error', (err) => {
    if (err.code === 'EADDRINUSE') {
        console.error(`Port ${PORT} is already in use`);
    } else if (err.code === 'EADDRNOTAVAIL') {
        console.error(`IP ${HOST} is not available on this machine`);
    } else {
        console.error(`Server error: ${err.message}`);
    }
    process.exit(1);
});
```

## Listen on All Interfaces vs Specific IP

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
    res.end('OK');
});

// Listen on ALL IPv4 interfaces (most common for production)
server.listen(8080, '0.0.0.0', () => {
    console.log('Listening on all interfaces: 0.0.0.0:8080');
});

// OR: Listen only on loopback (internal services only)
// server.listen(8080, '127.0.0.1', ...);

// OR: Listen on a specific NIC
// server.listen(8080, '10.0.0.5', ...);
```

## Express Bound to Specific IPv4 Address

```javascript
const express = require('express');

const app = express();
const HOST = '192.168.1.50';
const PORT = 3000;

app.use(express.json());

app.get('/health', (req, res) => {
    res.json({ status: 'ok', server: HOST });
});

app.get('/api/data', (req, res) => {
    // req.ip gives the client IP (respects X-Forwarded-For if trust proxy set)
    res.json({ message: 'Data', clientIP: req.ip });
});

// Pass host as second argument to app.listen()
const server = app.listen(PORT, HOST, () => {
    console.log(`Express server on http://${HOST}:${PORT}`);
});

// Graceful shutdown
process.on('SIGTERM', () => {
    server.close(() => {
        console.log('Server closed');
        process.exit(0);
    });
});
```

## Detecting Available IPv4 Addresses

```javascript
const os = require('os');

function getIPv4Addresses() {
    const interfaces = os.networkInterfaces();
    const addresses = [];

    for (const [name, iface] of Object.entries(interfaces)) {
        for (const addr of iface) {
            if (addr.family === 'IPv4' && !addr.internal) {
                addresses.push({ interface: name, ip: addr.address });
            }
        }
    }

    return addresses;
}

console.log('Available IPv4 addresses:');
getIPv4Addresses().forEach(({ interface: iface, ip }) => {
    console.log(`  ${iface}: ${ip}`);
});
```

## Verifying the Binding

```bash
# Check which IP the server is bound to

ss -tlnp | grep 8080

# Or with netstat
netstat -tlnp | grep 8080

# Test from another host
curl http://192.168.1.50:8080/health
```

## Conclusion

Binding a Node.js HTTP server to a specific IPv4 address is done by passing the host as the second argument to `server.listen(port, host, callback)`. Use `0.0.0.0` for all interfaces, `127.0.0.1` for loopback only, or a specific IP to restrict to a particular network interface. Always handle `EADDRNOTAVAIL` in case the specified IP doesn't exist on the machine.
