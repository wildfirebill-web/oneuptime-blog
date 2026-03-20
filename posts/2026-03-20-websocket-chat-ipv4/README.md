# How to Build a Real-Time Chat Application with WebSockets over IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WebSocket, IPv4, Chat, Python, JavaScript, Real-Time

Description: Learn how to build a real-time chat application using WebSockets over IPv4, with a Python asyncio server, browser JavaScript client, room support, and username handling.

## Server: Python asyncio + websockets

```python
import asyncio
import json
import logging
from dataclasses import dataclass, field
from typing import Optional
import websockets
from websockets.server import WebSocketServerProtocol

logging.basicConfig(level=logging.INFO)
log = logging.getLogger(__name__)

@dataclass
class Client:
    ws:       WebSocketServerProtocol
    username: str
    room:     str = "general"

clients: dict[WebSocketServerProtocol, Client] = {}

async def broadcast(room: str, message: dict, exclude: Optional[WebSocketServerProtocol] = None):
    payload = json.dumps(message)
    targets = [
        c for c in clients.values()
        if c.room == room and c.ws is not exclude
    ]
    await asyncio.gather(
        *[c.ws.send(payload) for c in targets],
        return_exceptions=True
    )

async def handler(ws: WebSocketServerProtocol):
    ip = ws.remote_address[0]

    # First message must be a join event with username
    try:
        raw  = await asyncio.wait_for(ws.recv(), timeout=10.0)
        data = json.loads(raw)
        if data.get("type") != "join" or not data.get("username"):
            await ws.close(4000, "Expected join message")
            return
    except (asyncio.TimeoutError, json.JSONDecodeError):
        await ws.close(4000, "Invalid join")
        return

    username = data["username"][:20]
    room     = data.get("room", "general")
    client   = Client(ws=ws, username=username, room=room)
    clients[ws] = client

    log.info("[+] %s (%s) joined #%s", username, ip, room)
    await broadcast(room, {"type": "system", "text": f"{username} joined"})

    try:
        async for raw in ws:
            try:
                msg = json.loads(raw)
            except json.JSONDecodeError:
                continue
            if msg.get("type") == "message":
                text = str(msg.get("text", ""))[:1000]
                log.info("[%s] %s: %s", room, username, text)
                await broadcast(room, {
                    "type":     "message",
                    "from":     username,
                    "text":     text,
                }, exclude=ws)
    except websockets.ConnectionClosed:
        pass
    finally:
        del clients[ws]
        log.info("[-] %s left #%s", username, room)
        await broadcast(room, {"type": "system", "text": f"{username} left"})

async def main():
    async with websockets.serve(handler, "0.0.0.0", 8765):
        log.info("Chat server on ws://0.0.0.0:8765")
        await asyncio.Future()

asyncio.run(main())
```

## Browser JavaScript Client

```html
<!DOCTYPE html>
<html>
<head><title>Chat</title></head>
<body>
<div id="messages"></div>
<input id="text" placeholder="Message..." />
<button onclick="send()">Send</button>

<script>
const ws = new WebSocket("ws://192.168.1.10:8765");

ws.onopen = () => {
    ws.send(JSON.stringify({ type: "join", username: "Alice", room: "general" }));
};

ws.onmessage = (e) => {
    const msg = JSON.parse(e.data);
    const div = document.createElement("div");
    if (msg.type === "message") {
        div.textContent = `${msg.from}: ${msg.text}`;
    } else {
        div.textContent = `*** ${msg.text} ***`;
        div.style.color = "gray";
    }
    document.getElementById("messages").appendChild(div);
};

function send() {
    const input = document.getElementById("text");
    ws.send(JSON.stringify({ type: "message", text: input.value }));
    input.value = "";
}

document.getElementById("text").addEventListener("keydown", (e) => {
    if (e.key === "Enter") send();
});
</script>
</body>
</html>
```

## Message Protocol

```json
// Client → Server (join)
{ "type": "join", "username": "Alice", "room": "general" }

// Client → Server (message)
{ "type": "message", "text": "Hello!" }

// Server → Client (message)
{ "type": "message", "from": "Alice", "text": "Hello!" }

// Server → Client (system)
{ "type": "system", "text": "Bob joined" }
```

## Conclusion

The chat application uses a simple join-then-message protocol over WebSocket. The server maintains a registry of `Client` objects keyed by WebSocket connection, enabling room-based filtering for broadcast. Broadcast uses `asyncio.gather` with `return_exceptions=True` to avoid one failing send blocking others. Input validation (username length cap, message length cap, type checking) is critical for public-facing chat applications. For production, add authentication, message persistence, and Redis pub/sub for multi-server deployments.
