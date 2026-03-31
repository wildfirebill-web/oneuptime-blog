# How to Handle Multiple TCP Connections in Node.js over IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Node.js, TCP, IPv4, Networking, Net Module, Concurrency, Socket

Description: Use the Node.js net module to build a TCP server that handles multiple simultaneous IPv4 client connections with proper event handling and connection lifecycle management.

## Introduction

Node.js's built-in `net` module provides low-level TCP socket functionality. Because Node.js uses an event-driven, non-blocking architecture, a single process can handle thousands of simultaneous TCP connections without threading. This guide shows how to build a robust multi-client TCP server over IPv4.

## Basic Multi-Client TCP Server

```javascript
const net = require('net');

// Track all connected clients
const clients = new Map();
let clientIdCounter = 0;

const server = net.createServer((socket) => {
  const clientId = ++clientIdCounter;
  const clientInfo = {
    id: clientId,
    address: socket.remoteAddress,
    port: socket.remotePort,
    socket: socket
  };
  
  clients.set(clientId, clientInfo);
  console.log(`Client ${clientId} connected from ${socket.remoteAddress}:${socket.remotePort}`);
  console.log(`Total clients: ${clients.size}`);
  
  // Send welcome message
  socket.write(`Welcome! You are client #${clientId}\n`);
  
  // Handle incoming data
  socket.on('data', (data) => {
    const message = data.toString().trim();
    console.log(`Client ${clientId}: ${message}`);
    
    // Echo back to sender
    socket.write(`Echo: ${message}\n`);
    
    // Broadcast to all other clients
    for (const [id, client] of clients) {
      if (id !== clientId && !client.socket.destroyed) {
        client.socket.write(`Client ${clientId}: ${message}\n`);
      }
    }
  });
  
  // Handle client disconnect
  socket.on('end', () => {
    console.log(`Client ${clientId} disconnected gracefully`);
    clients.delete(clientId);
  });
  
  // Handle connection errors
  socket.on('error', (err) => {
    console.error(`Client ${clientId} error: ${err.message}`);
    clients.delete(clientId);
  });
  
  // Handle socket close
  socket.on('close', (hadError) => {
    clients.delete(clientId);
    console.log(`Client ${clientId} closed. Remaining: ${clients.size}`);
  });
});

// Bind to IPv4 only
server.listen(8080, '0.0.0.0', () => {
  const addr = server.address();
  console.log(`TCP server listening on ${addr.address}:${addr.port}`);
});

// Handle server-level errors
server.on('error', (err) => {
  if (err.code === 'EADDRINUSE') {
    console.error('Port 8080 is already in use');
  } else {
    console.error(`Server error: ${err.message}`);
  }
  process.exit(1);
});
```

## Setting Connection Limits and Timeouts

```javascript
const server = net.createServer();

// Limit maximum number of concurrent connections
server.maxConnections = 1000;

server.on('connection', (socket) => {
  // Set inactivity timeout (30 seconds)
  socket.setTimeout(30000);
  
  socket.on('timeout', () => {
    console.log(`Client ${socket.remoteAddress} timed out - closing`);
    socket.end('Timeout: connection closed due to inactivity\n');
  });
  
  // Enable keep-alive probes
  socket.setKeepAlive(true, 60000); // Start keepalive after 60s idle
  
  // Disable Nagle's algorithm for low-latency messaging
  socket.setNoDelay(true);
  
  socket.on('data', (data) => {
    // Process data...
    socket.write(data); // Echo
  });
});
```

## Graceful Shutdown

```javascript
// Handle SIGTERM for graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received. Closing server...');
  
  // Stop accepting new connections
  server.close(() => {
    console.log('Server closed. No new connections accepted.');
  });
  
  // Close existing connections with a message
  for (const client of clients.values()) {
    if (!client.socket.destroyed) {
      client.socket.end('Server is shutting down\n');
    }
  }
  
  // Force close after 10 seconds
  setTimeout(() => {
    console.log('Force closing remaining connections');
    for (const client of clients.values()) {
      client.socket.destroy();
    }
    process.exit(0);
  }, 10000);
});
```

## Monitoring Connection Count

```javascript
// Log connection statistics every 30 seconds
setInterval(() => {
  server.getConnections((err, count) => {
    if (!err) {
      console.log(`Active connections: ${count}`);
    }
  });
}, 30000);
```

## Conclusion

Node.js's event-driven TCP server naturally handles thousands of concurrent connections with a single thread. The key patterns are: tracking clients in a `Map`, handling `error`, `end`, `close`, and `timeout` events on each socket, and implementing graceful shutdown to avoid abruptly terminating client sessions.
