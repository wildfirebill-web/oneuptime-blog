# How to Secure WebSocket Connections with TLS over IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WebSocket, TLS, IPv4, Security, Python, Node.js

Description: Learn how to secure WebSocket connections with TLS (wss://) over IPv4 using Python, Node.js, and Nginx, enabling encrypted communication for browser and server WebSocket clients.

## Python: wss:// Server with ssl module

```python
import asyncio
import ssl
import websockets

def create_ssl_context() -> ssl.SSLContext:
    ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
    ctx.load_cert_chain("server.crt", "server.key")
    # Optionally require client certificate (mTLS):
    # ctx.verify_mode = ssl.CERT_REQUIRED
    # ctx.load_verify_locations("ca.crt")
    return ctx

async def handler(websocket):
    async for msg in websocket:
        await websocket.send(f"secure echo: {msg}")

async def main():
    ssl_ctx = create_ssl_context()
    async with websockets.serve(handler, "0.0.0.0", 8443, ssl=ssl_ctx):
        print("Secure WebSocket server on wss://0.0.0.0:8443")
        await asyncio.Future()

asyncio.run(main())
```

## Python: wss:// Client

```python
import asyncio
import ssl
import websockets

async def client():
    ctx = ssl.create_default_context()
    ctx.load_verify_locations("ca.crt")  # for self-signed CA

    async with websockets.connect("wss://192.168.1.10:8443", ssl=ctx) as ws:
        await ws.send("Hello, secure server!")
        reply = await ws.recv()
        print(reply)

asyncio.run(client())
```

## Node.js: HTTPS + wss:// Server

```javascript
const https    = require("https");
const WebSocket = require("ws");
const fs       = require("fs");

const server = https.createServer({
    key:  fs.readFileSync("server.key"),
    cert: fs.readFileSync("server.crt"),
    // ca: fs.readFileSync("ca.crt"),        // for mTLS
    // requestCert: true,                    // require client cert
    // rejectUnauthorized: true,
});

const wss = new WebSocket.Server({ server });

wss.on("connection", (ws, req) => {
    const ip = req.socket.remoteAddress.replace("::ffff:", "");
    console.log(`Secure connection from ${ip}`);

    ws.on("message", (data) => ws.send(`secure echo: ${data}`));
});

server.listen(8443, "0.0.0.0", () => {
    console.log("wss://0.0.0.0:8443");
});
```

## Nginx: TLS Termination for WebSocket

```nginx
server {
    listen 443 ssl;
    server_name ws.example.com;

    ssl_certificate     /etc/ssl/server.crt;
    ssl_certificate_key /etc/ssl/server.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location /ws {
        proxy_pass http://127.0.0.1:8765;  # plain ws:// backend

        # WebSocket upgrade headers
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host       $host;
        proxy_set_header X-Real-IP  $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_read_timeout 3600s;   # keep WebSocket connections alive
    }
}
```

## Browser Client Connecting to wss://

```javascript
// wss:// automatically uses TLS — no extra config needed in the browser
const ws = new WebSocket("wss://ws.example.com/ws");

ws.onopen = () => ws.send("hello over TLS!");
ws.onmessage = (e) => console.log(e.data);
```

## Conclusion

WebSocket over TLS uses the `wss://` scheme and defaults to port 443. The simplest production setup is to run the WebSocket server as plain `ws://` bound to `127.0.0.1` and let Nginx handle TLS termination with the `Upgrade`/`Connection` proxy headers. For self-signed certificates, provide the CA bundle to the client. Set `proxy_read_timeout` to a large value (3600s) to prevent Nginx from closing idle WebSocket connections.
