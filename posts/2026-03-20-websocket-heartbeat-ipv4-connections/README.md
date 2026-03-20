# How to Implement WebSocket Heartbeat over IPv4 Connections

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WebSocket, Heartbeat, IPv4, JavaScript, Node.js, Ping-Pong, Real-Time

Description: Implement WebSocket ping/pong heartbeat mechanisms over IPv4 connections in Node.js to detect dead connections and maintain persistent WebSocket sessions.

## Introduction

WebSocket connections can silently die when a network path becomes unreachable - firewalls drop idle connections, NAT mappings expire, or intermediate routers fail. A heartbeat (ping/pong) mechanism detects these dead connections before your application tries to use them.

## WebSocket Ping/Pong Protocol

The WebSocket spec defines `ping` (opcode 0x9) and `pong` (opcode 0xA) control frames. The server sends a `ping`; the client must immediately reply with a `pong`. If no pong is received within a timeout, the connection is considered dead.

## Server-Side Heartbeat (Node.js with ws)

```javascript
// server.js
const WebSocket = require('ws');

const PORT = 8080;
const HEARTBEAT_INTERVAL = 30000;  // Send ping every 30 seconds
const HEARTBEAT_TIMEOUT = 10000;   // Expect pong within 10 seconds

const wss = new WebSocket.Server({
  host: '0.0.0.0',           // Listen on all IPv4 interfaces
  port: PORT
});

console.log(`WebSocket server on ws://0.0.0.0:${PORT}`);

function heartbeat(ws) {
  ws.isAlive = true;  // Reset on pong received
}

wss.on('connection', (ws, req) => {
  const clientIP = req.socket.remoteAddress;
  console.log(`Client connected: ${clientIP}`);
  
  // Initialize the liveness flag
  ws.isAlive = true;
  
  // Respond to pong frames (sent by client in response to our ping)
  ws.on('pong', () => {
    heartbeat(ws);
    console.log(`Pong received from ${clientIP}`);
  });
  
  ws.on('message', (data) => {
    console.log(`Message from ${clientIP}: ${data}`);
    ws.send(`Echo: ${data}`);
  });
  
  ws.on('close', (code, reason) => {
    console.log(`Client ${clientIP} disconnected: ${code} ${reason}`);
  });
  
  ws.on('error', (err) => {
    console.error(`Error from ${clientIP}: ${err.message}`);
  });
});

// Periodic heartbeat check: ping all connected clients
const heartbeatTimer = setInterval(() => {
  wss.clients.forEach((ws) => {
    if (!ws.isAlive) {
      // No pong received since last ping - terminate the connection
      console.log('Terminating dead connection (no pong received)');
      ws.terminate();
      return;
    }
    
    // Mark as potentially dead, send ping
    ws.isAlive = false;
    ws.ping('', false, (err) => {
      if (err) {
        console.error(`Ping failed: ${err.message}`);
      }
    });
  });
}, HEARTBEAT_INTERVAL);

// Clean up on server close
wss.on('close', () => {
  clearInterval(heartbeatTimer);
});
```

## Client-Side Heartbeat

```javascript
// client.js
const WebSocket = require('ws');

const SERVER_URL = 'ws://192.168.1.100:8080';
const CLIENT_PING_INTERVAL = 25000;   // Client-initiated ping every 25 seconds
const PONG_TIMEOUT = 10000;           // Expect pong within 10 seconds

function createConnection() {
  const ws = new WebSocket(SERVER_URL);
  let pingTimer = null;
  let pongTimeout = null;
  
  ws.on('open', () => {
    console.log('Connected to server');
    startHeartbeat(ws);
  });
  
  ws.on('ping', () => {
    // Automatically handled by the ws library - sends pong immediately
    console.log('Received ping from server - pong sent automatically');
  });
  
  ws.on('pong', () => {
    // Our ping was answered - clear the pong timeout
    clearTimeout(pongTimeout);
    console.log('Pong received from server - connection is alive');
  });
  
  ws.on('message', (data) => {
    console.log(`Server: ${data}`);
  });
  
  ws.on('close', (code, reason) => {
    console.log(`Disconnected: ${code}`);
    clearTimeout(pingTimer);
    clearTimeout(pongTimeout);
    
    // Reconnect after 5 seconds
    console.log('Reconnecting in 5 seconds...');
    setTimeout(createConnection, 5000);
  });
  
  ws.on('error', (err) => {
    console.error(`WebSocket error: ${err.message}`);
  });
  
  function startHeartbeat(ws) {
    pingTimer = setInterval(() => {
      if (ws.readyState === WebSocket.OPEN) {
        ws.ping('heartbeat');
        
        // If pong is not received within timeout, close the connection
        pongTimeout = setTimeout(() => {
          console.log('Pong timeout - connection appears dead, closing');
          ws.terminate();
        }, PONG_TIMEOUT);
      }
    }, CLIENT_PING_INTERVAL);
  }
  
  return ws;
}

const ws = createConnection();

// Send test messages every 5 seconds
setInterval(() => {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(`Hello at ${new Date().toISOString()}`);
  }
}, 5000);
```

## Browser WebSocket Heartbeat

For browser clients (where you can't send custom pings):

```javascript
// browser-client.js
class HeartbeatWebSocket {
  constructor(url) {
    this.url = url;
    this.pingInterval = null;
    this.pongTimeout = null;
    this.connect();
  }
  
  connect() {
    this.ws = new WebSocket(this.url);
    
    this.ws.onopen = () => {
      console.log('Connected');
      this.startHeartbeat();
    };
    
    this.ws.onmessage = (event) => {
      // Handle application-level pong response
      if (event.data === '__pong__') {
        clearTimeout(this.pongTimeout);
        return;
      }
      console.log('Message:', event.data);
    };
    
    this.ws.onclose = () => {
      this.cleanup();
      setTimeout(() => this.connect(), 3000);
    };
  }
  
  startHeartbeat() {
    // Send application-level ping (since browser can't send WS ping frames)
    this.pingInterval = setInterval(() => {
      if (this.ws.readyState === WebSocket.OPEN) {
        this.ws.send('__ping__');
        
        this.pongTimeout = setTimeout(() => {
          console.log('Heartbeat failed - reconnecting');
          this.ws.close();
        }, 5000);
      }
    }, 20000);
  }
  
  cleanup() {
    clearInterval(this.pingInterval);
    clearTimeout(this.pongTimeout);
  }
}

const wsClient = new HeartbeatWebSocket('ws://192.168.1.100:8080');
```

## Conclusion

WebSocket heartbeats are essential for maintaining reliable long-lived connections over IPv4. Use the built-in WS ping/pong frames (RFC 6455) for native client support, and fall back to application-level `__ping__`/`__pong__` messages for browser clients that cannot send raw ping frames. Always implement automatic reconnection for client-side robustness.
