# How to Implement WebSocket Authentication with IPv4 Client Tracking

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WebSocket, IPv4, Authentication, Python, Node.js, Security

Description: Learn how to authenticate WebSocket connections using tokens and track client IPv4 addresses for access control, rate limiting, and security monitoring.

## Token Authentication on Handshake (Python)

```python
import asyncio
import websockets
from websockets.server import WebSocketServerProtocol
from websockets.exceptions import ConnectionClosed

# Simple token store — use a database or JWT in production
VALID_TOKENS = {"secret-token-abc": "user1", "secret-token-xyz": "user2"}

connected: dict[WebSocketServerProtocol, dict] = {}

async def handler(websocket: WebSocketServerProtocol, path: str):
    # Validate token sent as query parameter: ws://host:port/?token=xxx
    token = websocket.request_headers.get("Authorization", "").removeprefix("Bearer ")
    # Also check query string
    from urllib.parse import urlparse, parse_qs
    params = parse_qs(urlparse(path).query)
    token = token or (params.get("token", [""])[0])

    user = VALID_TOKENS.get(token)
    if not user:
        await websocket.close(4001, "Unauthorized")
        return

    ip = websocket.remote_address[0]
    connected[websocket] = {"user": user, "ip": ip}
    print(f"[+] {user} from {ip}")

    try:
        async for msg in websocket:
            await websocket.send(f"[{user}] echo: {msg}")
    except ConnectionClosed:
        pass
    finally:
        del connected[websocket]
        print(f"[-] {user} from {ip}")

async def main():
    async with websockets.serve(handler, "0.0.0.0", 8765):
        print("Authenticated WebSocket server on ws://0.0.0.0:8765")
        await asyncio.Future()

asyncio.run(main())
```

## Node.js: Token Auth + IP Tracking

```javascript
const WebSocket = require("ws");
const url      = require("url");

const VALID_TOKENS = new Map([
    ["secret-token-abc", "user1"],
    ["secret-token-xyz", "user2"],
]);

const clients = new Map();  // ws → { user, ip }

const wss = new WebSocket.Server({ host: "0.0.0.0", port: 8765 });

wss.on("connection", (ws, req) => {
    const ip     = req.socket.remoteAddress.replace("::ffff:", "");
    const params = new URLSearchParams(url.parse(req.url).query);
    const token  = params.get("token") ||
                   (req.headers.authorization || "").replace("Bearer ", "");

    const user = VALID_TOKENS.get(token);
    if (!user) {
        ws.close(4001, "Unauthorized");
        return;
    }

    clients.set(ws, { user, ip });
    console.log(`[+] ${user} from ${ip}`);

    ws.on("message", (data) => {
        console.log(`[${user}@${ip}] ${data}`);
        ws.send(`[${user}] echo: ${data}`);
    });

    ws.on("close", () => {
        console.log(`[-] ${user} from ${ip}`);
        clients.delete(ws);
    });
});
```

## IP-Based Rate Limiting for WebSockets

```python
import time
from collections import defaultdict

# Sliding window rate limit per IP
_windows: dict[str, list[float]] = defaultdict(list)
LIMIT, WINDOW = 10, 1.0  # 10 messages per second

def is_rate_limited(ip: str) -> bool:
    now = time.monotonic()
    _windows[ip] = [t for t in _windows[ip] if t > now - WINDOW]
    if len(_windows[ip]) >= LIMIT:
        return True
    _windows[ip].append(now)
    return False

async def handler(websocket):
    ip = websocket.remote_address[0]
    try:
        async for msg in websocket:
            if is_rate_limited(ip):
                await websocket.close(4029, "Rate limit exceeded")
                return
            await websocket.send(f"echo: {msg}")
    except websockets.ConnectionClosed:
        pass
```

## Conclusion

Authenticate WebSocket connections during the HTTP upgrade handshake — the `request_headers` (Python `websockets`) and `req.url`/`req.headers` (Node.js `ws`) are available before the WebSocket protocol takes over. Store authenticated session info (user, IP) in a client registry for access control and audit logging. Implement per-IP rate limiting on incoming messages to protect against abusive clients. Close with application error codes in the 4000-4999 range to distinguish auth failures from protocol errors.
