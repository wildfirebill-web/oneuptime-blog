# How to Connect a WebSocket Client to an IPv4 Server Address

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WebSocket, IPv4, Python, JavaScript, Go, Client

Description: Learn how to connect WebSocket clients to IPv4 server addresses in Python, JavaScript, and Go, with reconnection logic, message handling, and TLS support.

## Python: websockets Client

```python
import asyncio
import websockets

async def client():
    # Connect to a specific IPv4 address
    uri = "ws://192.168.1.10:8765"
    async with websockets.connect(uri) as ws:
        print(f"Connected to {uri}")

        # Send a message
        await ws.send("Hello, server!")
        reply = await ws.recv()
        print(f"Server replied: {reply}")

asyncio.run(client())
```

## Python: Persistent Client with Reconnection

```python
import asyncio
import websockets
import logging

log = logging.getLogger(__name__)

async def client_loop(uri: str) -> None:
    backoff = 1
    while True:
        try:
            async with websockets.connect(uri, ping_interval=20) as ws:
                log.info("Connected to %s", uri)
                backoff = 1  # reset on successful connection
                async for message in ws:
                    log.info("Received: %s", message)
                    # process message ...
        except (websockets.ConnectionClosed, OSError) as e:
            log.warning("Disconnected: %s — retry in %ds", e, backoff)
            await asyncio.sleep(backoff)
            backoff = min(backoff * 2, 60)  # exponential backoff, cap at 60s

asyncio.run(client_loop("ws://192.168.1.10:8765"))
```

## Browser JavaScript

```javascript
const ws = new WebSocket("ws://192.168.1.10:8765");

ws.addEventListener("open", () => {
    console.log("Connected");
    ws.send(JSON.stringify({ type: "hello" }));
});

ws.addEventListener("message", (event) => {
    const data = JSON.parse(event.data);
    console.log("Received:", data);
});

ws.addEventListener("close", (event) => {
    console.log(`Closed: code=${event.code} reason=${event.reason}`);
});

ws.addEventListener("error", (err) => {
    console.error("WebSocket error:", err);
});
```

## Node.js: ws Client

```javascript
const WebSocket = require("ws");

function connect(url) {
    const ws = new WebSocket(url);
    let reconnectDelay = 1000;

    ws.on("open", () => {
        console.log("Connected to", url);
        reconnectDelay = 1000;
        ws.send(JSON.stringify({ type: "hello" }));
    });

    ws.on("message", (data) => {
        console.log("Message:", data.toString());
    });

    ws.on("close", (code, reason) => {
        console.log(`Closed (${code}): ${reason} — reconnecting in ${reconnectDelay}ms`);
        setTimeout(() => connect(url), reconnectDelay);
        reconnectDelay = Math.min(reconnectDelay * 2, 60_000);
    });

    ws.on("error", (err) => console.error("Error:", err.message));
}

connect("ws://192.168.1.10:8765");
```

## Go: WebSocket Client

```go
package main

import (
    "log"
    "net/url"
    "time"
    "github.com/gorilla/websocket"
)

func main() {
    u := url.URL{Scheme: "ws", Host: "192.168.1.10:8765", Path: "/ws"}
    log.Printf("Connecting to %s", u.String())

    conn, _, err := websocket.DefaultDialer.Dial(u.String(), nil)
    if err != nil {
        log.Fatal("dial:", err)
    }
    defer conn.Close()

    // Send
    conn.WriteMessage(websocket.TextMessage, []byte("Hello, server!"))

    // Receive
    conn.SetReadDeadline(time.Now().Add(10 * time.Second))
    _, msg, err := conn.ReadMessage()
    if err != nil {
        log.Fatal("read:", err)
    }
    log.Printf("Server: %s", msg)
}
```

## Conclusion

WebSocket client URLs use `ws://` for plain-text or `wss://` for TLS. The IPv4 server address goes directly in the URL host: `ws://192.168.1.10:8765`. Always implement reconnection logic with exponential backoff — the `async for message in ws` (Python) pattern cleanly handles disconnections via `ConnectionClosed` exceptions. In the browser, attach event listeners to `open`, `message`, `close`, and `error` events and rebuild the connection in the `close` handler.
