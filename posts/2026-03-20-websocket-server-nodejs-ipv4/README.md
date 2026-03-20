# How to Create a WebSocket Server on IPv4 in Node.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WebSocket, Node.js, IPv4, JavaScript, Networking, Ws

Description: Learn how to create a WebSocket server on a specific IPv4 address in Node.js using the ws library, with connection tracking, broadcasting, heartbeat, and graceful shutdown.

## Installation

```bash
npm install ws
```

## Basic WebSocket Server

```javascript
const WebSocket = require("ws");

const wss = new WebSocket.Server({
    host: "0.0.0.0",  // bind to all IPv4 interfaces
    port: 8765,
});

wss.on("listening", () => {
    const { address, port } = wss.address();
    console.log(`WebSocket server on ws://${address}:${port}`);
});

wss.on("connection", (ws, req) => {
    const ip = req.socket.remoteAddress.replace("::ffff:", "");
    console.log(`[+] Client connected: ${ip}`);

    ws.on("message", (data) => {
        console.log(`Message from ${ip}: ${data}`);
        ws.send(`Echo: ${data}`);
    });

    ws.on("close", () => console.log(`[-] Client disconnected: ${ip}`));
    ws.on("error", (err) => console.error(`Error (${ip}): ${err.message}`));
});
```

## Broadcast to All Clients

```javascript
const WebSocket = require("ws");

const wss = new WebSocket.Server({ host: "0.0.0.0", port: 8765 });

function broadcast(data, exclude = null) {
    const payload = typeof data === "object" ? JSON.stringify(data) : data;
    for (const client of wss.clients) {
        if (client !== exclude && client.readyState === WebSocket.OPEN) {
            client.send(payload);
        }
    }
}

wss.on("connection", (ws, req) => {
    const ip = req.socket.remoteAddress.replace("::ffff:", "");
    ws.ip = ip;

    broadcast({ type: "join", ip });

    ws.on("message", (data) => {
        try {
            const msg = JSON.parse(data);
            broadcast({ type: "message", from: ip, ...msg }, ws);
        } catch {
            ws.send(JSON.stringify({ error: "invalid JSON" }));
        }
    });

    ws.on("close", () => broadcast({ type: "leave", ip }));
});
```

## Heartbeat (Detect Stale Connections)

```javascript
const WebSocket = require("ws");

const wss = new WebSocket.Server({ host: "0.0.0.0", port: 8765 });

wss.on("connection", (ws) => {
    ws.isAlive = true;
    ws.on("pong", () => { ws.isAlive = true; });

    ws.on("message", (data) => ws.send(`echo: ${data}`));
});

// Ping all clients every 30 seconds; terminate unresponsive ones
const heartbeat = setInterval(() => {
    for (const ws of wss.clients) {
        if (!ws.isAlive) {
            ws.terminate();
            continue;
        }
        ws.isAlive = false;
        ws.ping();
    }
}, 30_000);

wss.on("close", () => clearInterval(heartbeat));
```

## Graceful Shutdown

```javascript
const WebSocket = require("ws");

const wss = new WebSocket.Server({ host: "0.0.0.0", port: 8765 });

process.on("SIGTERM", () => {
    console.log("Shutting down WebSocket server");

    // Notify all clients
    for (const ws of wss.clients) {
        ws.close(1001, "Server shutting down");
    }

    wss.close(() => {
        console.log("Server closed");
        process.exit(0);
    });

    setTimeout(() => process.exit(1), 10_000);
});
```

## Conclusion

The `ws` library's `WebSocket.Server` accepts `host` and `port` options to bind to a specific IPv4 address. Use `wss.clients` to iterate all connected clients and `ws.readyState === WebSocket.OPEN` to guard sends. Implement a ping/pong heartbeat to detect clients that disconnect without sending a close frame (e.g., network failure, browser tab close). Store the client IP from `req.socket.remoteAddress` (stripping the `::ffff:` prefix) for logging and access control.
