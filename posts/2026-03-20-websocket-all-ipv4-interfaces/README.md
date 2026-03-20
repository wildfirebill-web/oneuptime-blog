# How to Configure WebSocket to Listen on All IPv4 Interfaces (0.0.0.0)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WebSocket, IPv4, Python, Node.js, Go, Networking

Description: Learn how to configure WebSocket servers to listen on all IPv4 interfaces using 0.0.0.0, understand the implications for containerised deployments, and expose servers securely.

## Why 0.0.0.0 for WebSocket Servers

| Bind Address | Accessible From |
|---|---|
| `127.0.0.1` | Same machine only |
| `192.168.1.10` | That specific interface only |
| `0.0.0.0` | All IPv4 interfaces (all clients on the network) |

In Docker/Kubernetes, `0.0.0.0` is required so that traffic routed through the virtual NIC reaches the server.

## Python: websockets on 0.0.0.0

```python
import asyncio
import os
import websockets

async def handler(ws):
    async for msg in ws:
        await ws.send(f"echo: {msg}")

async def main():
    host = os.environ.get("WS_HOST", "0.0.0.0")
    port = int(os.environ.get("WS_PORT", "8765"))
    async with websockets.serve(handler, host, port):
        print(f"WebSocket server on ws://{host}:{port}")
        await asyncio.Future()

asyncio.run(main())
```

## Node.js: ws on 0.0.0.0

```javascript
const WebSocket = require("ws");
const wss = new WebSocket.Server({
    host: process.env.WS_HOST || "0.0.0.0",
    port: parseInt(process.env.WS_PORT || "8765"),
});

wss.on("listening", () => {
    const { address, port } = wss.address();
    console.log(`WebSocket server on ws://${address}:${port}`);
});

wss.on("connection", (ws, req) => {
    const ip = req.socket.remoteAddress.replace("::ffff:", "");
    ws.on("message", (data) => ws.send(`echo: ${data}`));
});
```

## Go: Bind to 0.0.0.0 Explicitly

```go
package main

import (
    "fmt"
    "log"
    "net"
    "net/http"
    "os"

    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{CheckOrigin: func(*http.Request) bool { return true }}

func wsHandler(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil { return }
    defer conn.Close()
    for {
        mt, msg, err := conn.ReadMessage()
        if err != nil { break }
        conn.WriteMessage(mt, msg)
    }
}

func main() {
    host := os.Getenv("WS_HOST")
    if host == "" { host = "0.0.0.0" }
    port := os.Getenv("WS_PORT")
    if port == "" { port = "8765" }
    addr := host + ":" + port

    http.HandleFunc("/ws", wsHandler)

    // tcp4 = IPv4 only
    ln, err := net.Listen("tcp4", addr)
    if err != nil { log.Fatal(err) }

    fmt.Printf("WebSocket server on ws://%s/ws\n", addr)
    log.Fatal(http.Serve(ln, nil))
}
```

## Docker: Expose Port

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install websockets
COPY server.py .
EXPOSE 8765
CMD ["python", "server.py"]
```

```yaml
# docker-compose.yml
services:
  ws-server:
    build: .
    ports:
      - "8765:8765"     # host:container
    environment:
      WS_HOST: "0.0.0.0"
      WS_PORT: "8765"
```

## Security Note

Binding to `0.0.0.0` exposes the server on all interfaces. In production:
- Place a reverse proxy (Nginx) in front and bind WebSocket server to `127.0.0.1`
- Use `wss://` (WebSocket over TLS) for connections from untrusted networks
- Implement authentication in the WebSocket handshake

## Conclusion

Use `host="0.0.0.0"` (Python/Node.js) or `net.Listen("tcp4", "0.0.0.0:port")` (Go) for container and multi-host deployments where traffic arrives on the virtual NIC. Externalise the bind address via environment variables to allow restriction to `127.0.0.1` behind a reverse proxy without code changes. Never expose raw WebSocket servers directly to the internet — place an Nginx proxy in front for TLS termination and rate limiting.
