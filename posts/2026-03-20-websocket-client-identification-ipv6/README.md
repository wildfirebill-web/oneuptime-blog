# How to Handle IPv6 Client Identification in WebSocket

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WebSocket, IPv6, Authentication, Security, Node.js

Description: Extract, normalize, and use IPv6 client addresses in WebSocket servers for authentication, logging, and per-client session management.

## Overview

Extract, normalize, and use IPv6 client addresses in WebSocket servers for authentication, logging, and per-client session management. This guide covers the essential configuration and best practices for IPv6 compatibility.

## Prerequisites

- Basic knowledge of WebSocket protocol
- IPv6 connectivity on your server
- The relevant software or framework installed

## IPv6 WebSocket Fundamentals

WebSocket connections are HTTP upgrades. For IPv6 support, ensure:
1. Your server binds to `::` (all IPv6 interfaces)
2. Firewalls allow TCP on the WebSocket port over IPv6
3. Client URLs use bracketed IPv6 addresses: `ws://[::1]:8080/`

## Configuration

### Server Binding

```javascript
// Node.js ws library - bind to IPv6
const WebSocket = require('ws');
const wss = new WebSocket.Server({ host: '::', port: 8080 });

wss.on('connection', (ws, req) => {
    // Get client IPv6 address
    const clientIP = req.socket.remoteAddress;
    console.log('Client:', clientIP);
});
```

### Firewall Rules

```bash
# Allow WebSocket port over IPv6

sudo ip6tables -A INPUT -p tcp --dport 8080 -j ACCEPT -m comment --comment "WebSocket IPv6"

# Verify the rule is in place
sudo ip6tables -L INPUT -v -n | grep 8080
```

### Client Connection

```javascript
// Browser or Node.js client - IPv6 address needs brackets
const socket = new WebSocket('ws://[2001:db8::1]:8080/');

socket.addEventListener('open', () => {
    socket.send('Hello over IPv6!');
});

socket.addEventListener('message', (event) => {
    console.log('Received:', event.data);
});
```

## Testing

```bash
# Test WebSocket connectivity over IPv6
npm install -g wscat
wscat -c ws://[::1]:8080/

# Verify server is listening on IPv6
sudo ss -tlnp | grep 8080

# Check for IPv6 in access logs
tail -f /var/log/nginx/access.log | grep "::"
```

## Monitoring with OneUptime

Use [OneUptime](https://oneuptime.com) to monitor WebSocket endpoint availability over IPv6. Create TCP port monitors for your WebSocket port at your IPv6 address and configure alerts for connection failures or elevated error rates.

## Conclusion

How to Handle IPv6 Client Identification in WebSocket requires binding to IPv6 interfaces, configuring firewalls, and using correct IPv6 URL format for clients. Test thoroughly with wscat and automate IPv6-specific connectivity tests in your CI/CD pipeline.
