# How to Monitor WebSocket Connection Health over IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WebSocket, IPv4, Monitoring, Health Check, Python, Node.js

Description: Learn how to monitor WebSocket connection health over IPv4 using ping/pong keepalives, connection counters, Prometheus metrics, and external health check probes.

## Server-Side Connection Metrics (Python)

```python
import asyncio
import time
import websockets
from dataclasses import dataclass, field
from typing import Set

@dataclass
class Stats:
    total_connected:    int = 0
    total_disconnected: int = 0
    message_count:      int = 0
    error_count:        int = 0
    start_time:         float = field(default_factory=time.monotonic)

stats   = Stats()
clients: Set = set()

async def handler(ws):
    stats.total_connected += 1
    clients.add(ws)
    try:
        async for msg in ws:
            stats.message_count += 1
            await ws.send(msg)
    except websockets.ConnectionClosed:
        pass
    except Exception:
        stats.error_count += 1
    finally:
        clients.discard(ws)
        stats.total_disconnected += 1

# Health endpoint served over regular HTTP
from aiohttp import web

async def health(request):
    uptime = time.monotonic() - stats.start_time
    return web.json_response({
        "status":      "ok",
        "connections": len(clients),
        "uptime_s":    round(uptime, 1),
        "messages":    stats.message_count,
        "errors":      stats.error_count,
    })

async def main():
    # WebSocket server
    ws_server  = websockets.serve(handler, "0.0.0.0", 8765)
    # HTTP health server
    app  = web.Application()
    app.router.add_get("/health", health)
    runner = web.AppRunner(app)
    await runner.setup()
    site = web.TCPSite(runner, "127.0.0.1", 8766)

    async with ws_server:
        await site.start()
        print("WS on :8765  health on :8766/health")
        await asyncio.Future()

asyncio.run(main())
```

## Node.js: Prometheus Metrics with prom-client

```javascript
const WebSocket = require("ws");
const http      = require("http");
const client    = require("prom-client");

const register = new client.Registry();
client.collectDefaultMetrics({ register });

const wsConnections = new client.Gauge({
    name: "ws_active_connections",
    help: "Active WebSocket connections",
    registers: [register],
});
const wsMessages = new client.Counter({
    name: "ws_messages_total",
    help: "Total WebSocket messages received",
    registers: [register],
});

const wss = new WebSocket.Server({ host: "0.0.0.0", port: 8765 });

wss.on("connection", (ws) => {
    wsConnections.inc();
    ws.on("message", () => wsMessages.inc());
    ws.on("close",   () => wsConnections.dec());
});

// Prometheus scrape endpoint
const metricsServer = http.createServer(async (req, res) => {
    if (req.url === "/metrics") {
        res.setHeader("Content-Type", register.contentType);
        res.end(await register.metrics());
    } else {
        res.writeHead(404);
        res.end();
    }
});

metricsServer.listen(9090, "127.0.0.1", () => {
    console.log("Metrics on http://127.0.0.1:9090/metrics");
});
wss.on("listening", () => console.log("WS on ws://0.0.0.0:8765"));
```

## External Health Probe (Python)

```python
import asyncio
import websockets
import sys

async def probe(uri: str) -> bool:
    """Connect, send a ping, expect an echo. Returns True if healthy."""
    try:
        async with websockets.connect(uri, open_timeout=3.0) as ws:
            await ws.send("health-check")
            reply = await asyncio.wait_for(ws.recv(), timeout=3.0)
            return reply == "health-check"
    except Exception as e:
        print(f"Health check failed: {e}", file=sys.stderr)
        return False

async def main():
    ok = await probe("ws://192.168.1.10:8765")
    sys.exit(0 if ok else 1)

asyncio.run(main())
```

## Conclusion

Expose a separate HTTP health endpoint alongside the WebSocket server — Kubernetes liveness probes and monitoring systems can query it without maintaining a WebSocket connection. Export Prometheus metrics for active connection count, message throughput, and error rates. Use server-side ping/pong (`ping_interval` in `websockets.serve`) to detect dead connections and clean up the client registry. Build an external probe that connects, sends a test message, and verifies the response for end-to-end health verification.
