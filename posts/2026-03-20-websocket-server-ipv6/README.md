# How to Configure WebSocket Servers with IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WebSocket, IPv6, Node.js, Real-time, Networking

Description: Configure WebSocket servers in Node.js and Python to accept connections over IPv6, including proper address handling and dual-stack configuration.

## WebSocket over IPv6 Basics

WebSocket connections are initiated via HTTP(S) upgrade requests. IPv6 support in WebSocket servers follows the same pattern as HTTP servers: bind to `::` for all IPv6 interfaces.

## Node.js ws Library

```javascript
// server-ws.js
const WebSocket = require('ws');

// Create WebSocket server on all IPv6 interfaces
const wss = new WebSocket.Server({
    host: '::',   // IPv6 wildcard — accepts both IPv4 and IPv6 on most systems
    port: 8080,
});

wss.on('listening', () => {
    const addr = wss.address();
    console.log(`WebSocket server listening on [${addr.address}]:${addr.port}`);
});

wss.on('connection', (ws, req) => {
    // Get client IPv6 address
    const clientIP = req.socket.remoteAddress;
    const isIPv6 = clientIP.includes(':');

    console.log(`New connection from ${clientIP} (${isIPv6 ? 'IPv6' : 'IPv4'})`);

    // Remove IPv4-mapped prefix if present (::ffff:x.x.x.x)
    const normalizedIP = clientIP.replace(/^::ffff:/, '');

    ws.on('message', (data) => {
        console.log(`Received from ${normalizedIP}: ${data}`);
        ws.send(`Echo from [${normalizedIP}]: ${data}`);
    });

    ws.on('close', () => {
        console.log(`Connection closed from ${normalizedIP}`);
    });
});
```

## Secure WebSocket (WSS) over IPv6

```javascript
// server-wss.js
const WebSocket = require('ws');
const https = require('https');
const fs = require('fs');

const httpsServer = https.createServer({
    cert: fs.readFileSync('/etc/ssl/certs/example.com.crt'),
    key: fs.readFileSync('/etc/ssl/private/example.com.key'),
});

const wss = new WebSocket.Server({ server: httpsServer });

// Listen on all IPv6 interfaces
httpsServer.listen(8443, '::', () => {
    console.log('Secure WebSocket server on [::]:8443');
});

wss.on('connection', (ws, req) => {
    const clientIP = req.socket.remoteAddress;
    console.log(`Secure connection from: ${clientIP}`);

    ws.send(JSON.stringify({
        type: 'connected',
        message: 'Welcome to the IPv6 WebSocket server',
        yourIP: clientIP,
    }));
});
```

## Python websockets Library

```python
# server.py
import asyncio
import websockets
import json

async def handler(websocket):
    """Handle incoming WebSocket connections over IPv6."""
    # Get client address — websocket.remote_address is (host, port) tuple
    # For IPv6: ('2001:db8::1', 12345, 0, 0)
    remote = websocket.remote_address
    client_ip = remote[0] if remote else 'unknown'

    print(f"New connection from {client_ip}")

    try:
        async for message in websocket:
            print(f"Received from {client_ip}: {message}")
            response = json.dumps({
                "type": "echo",
                "message": message,
                "from_ip": client_ip,
            })
            await websocket.send(response)
    except websockets.ConnectionClosed:
        print(f"Connection closed from {client_ip}")

async def main():
    # Listen on all IPv6 interfaces
    # host="::" binds to all IPv6 interfaces
    async with websockets.serve(handler, "::", 8080) as server:
        print("WebSocket server on [::]:8080")
        await asyncio.Future()  # Run forever

asyncio.run(main())
```

## Testing WebSocket over IPv6

```bash
# Install wscat for testing
npm install -g wscat

# Connect to WebSocket server over IPv6
wscat -c ws://[::1]:8080

# Connect to a remote IPv6 WebSocket server
wscat -c ws://[2001:db8::1]:8080

# Secure WebSocket over IPv6
wscat -c wss://[2001:db8::1]:8443

# With curl (WebSocket upgrade)
curl -6 --include \
     --no-buffer \
     --header "Connection: Upgrade" \
     --header "Upgrade: websocket" \
     --header "Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==" \
     --header "Sec-WebSocket-Version: 13" \
     http://[2001:db8::1]:8080/
```

## Firewall Configuration

```bash
# Allow WebSocket port 8080 over IPv6
sudo ip6tables -A INPUT -p tcp --dport 8080 -j ACCEPT

# Or with ufw
sudo ufw allow 8080/tcp

# For WSS (port 8443)
sudo ip6tables -A INPUT -p tcp --dport 8443 -j ACCEPT
```

## Monitoring with OneUptime

Use [OneUptime](https://oneuptime.com) to monitor WebSocket server availability by checking TCP connectivity to port 8080 over IPv6. For deeper monitoring, create a synthetic WebSocket connection test that sends a ping and verifies a pong response.

## Conclusion

WebSocket servers over IPv6 bind using `::` as the host address. In Node.js ws library, pass `host: '::'` to the constructor. In Python's websockets library, pass `"::"` as the host argument to `websockets.serve()`. Test with wscat using `ws://[ipv6addr]:port` URLs.
