# How to Handle WebSocket Reconnection over IPv4 Networks

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WebSocket, IPv4, Reconnection, JavaScript, Python, Resilience

Description: Learn how to implement robust WebSocket reconnection logic over IPv4 networks using exponential backoff, jitter, and state restoration to handle transient network failures gracefully.

## Browser JavaScript: Reconnecting WebSocket Class

```javascript
class ReconnectingWebSocket {
    constructor(url, options = {}) {
        this.url = url;
        this.minDelay    = options.minDelay    || 1000;
        this.maxDelay    = options.maxDelay    || 30000;
        this.jitter      = options.jitter      !== false;
        this.onmessage   = options.onmessage   || (() => {});
        this.onopen      = options.onopen      || (() => {});
        this.onclose     = options.onclose     || (() => {});
        this._delay      = this.minDelay;
        this._reconnect  = true;
        this.connect();
    }

    connect() {
        this.ws = new WebSocket(this.url);

        this.ws.onopen = (e) => {
            console.log("Connected to", this.url);
            this._delay = this.minDelay;  // reset backoff
            this.onopen(e);
        };

        this.ws.onmessage = (e) => this.onmessage(e);

        this.ws.onclose = (e) => {
            this.onclose(e);
            if (!this._reconnect) return;
            let delay = this._delay;
            if (this.jitter) delay += Math.random() * delay * 0.5;  // ±50% jitter
            console.log(`Reconnecting in ${Math.round(delay)}ms...`);
            setTimeout(() => this.connect(), delay);
            this._delay = Math.min(this._delay * 2, this.maxDelay);
        };

        this.ws.onerror = (e) => console.error("WebSocket error:", e);
    }

    send(data) {
        if (this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(data);
        }
    }

    close() {
        this._reconnect = false;
        this.ws.close();
    }
}

// Usage
const ws = new ReconnectingWebSocket("ws://192.168.1.10:8765", {
    onmessage: (e) => console.log("Received:", e.data),
    onopen:    ()  => ws.send("hello"),
});
```

## Python: Async Reconnection Loop

```python
import asyncio
import logging
import websockets

log = logging.getLogger(__name__)

async def connect_with_retry(uri: str) -> None:
    min_delay, max_delay = 1.0, 60.0
    delay = min_delay

    while True:
        try:
            async with websockets.connect(
                uri,
                ping_interval=20,
                ping_timeout=10,
                close_timeout=5,
            ) as ws:
                log.info("Connected to %s", uri)
                delay = min_delay  # reset on success
                async for message in ws:
                    log.info("Received: %s", message)
                    await ws.send(f"ack: {message}")

        except (websockets.ConnectionClosed, OSError, ConnectionRefusedError) as e:
            import random
            jitter = random.uniform(0, delay * 0.5)
            log.warning("Disconnected: %s - retry in %.1fs", e, delay + jitter)
            await asyncio.sleep(delay + jitter)
            delay = min(delay * 2, max_delay)

asyncio.run(connect_with_retry("ws://192.168.1.10:8765"))
```

## Node.js: Reconnection with State Restoration

```javascript
const WebSocket = require("ws");

let ws;
let userId = null;   // restore auth on reconnect

function connect() {
    ws = new WebSocket("ws://192.168.1.10:8765");

    ws.on("open", () => {
        console.log("Connected");
        // Re-authenticate after reconnect
        if (userId) {
            ws.send(JSON.stringify({ type: "auth", token: process.env.WS_TOKEN }));
        }
    });

    ws.on("message", (data) => {
        const msg = JSON.parse(data);
        if (msg.type === "auth_ok") {
            userId = msg.userId;
        }
        console.log("Received:", msg);
    });

    ws.on("close", () => {
        const delay = 1000 + Math.random() * 2000;  // 1-3 second jitter
        console.log(`Reconnecting in ${delay.toFixed(0)}ms`);
        setTimeout(connect, delay);
    });
}

connect();
```

## Conclusion

Implement exponential backoff starting at 1 second, doubling on each failure, capped at 30-60 seconds. Add random jitter (±50% of the delay) to prevent a thunderstorm of simultaneous reconnections after a server restart. Reset the delay to the minimum on a successful connection. Handle state restoration (authentication, subscriptions) in the `onopen` / `open` handler to ensure the session is valid after reconnection. Use server-side ping/pong to detect half-open TCP connections and trigger clean reconnects.
