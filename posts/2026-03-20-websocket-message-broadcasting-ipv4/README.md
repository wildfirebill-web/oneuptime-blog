# How to Implement WebSocket Message Broadcasting over IPv4

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: WebSocket, Broadcasting, IPv4, Node.js, Real-Time, Ws Library, Pub-Sub

Description: Build a WebSocket server in Node.js that broadcasts messages to all connected IPv4 clients with room-based filtering, binary support, and selective broadcasting.

## Introduction

Broadcasting is the core feature of real-time WebSocket applications - sending a message from one client to all (or a subset) of connected clients. This guide covers full broadcast, selective broadcast (rooms/channels), and filtered broadcasting patterns.

## Full Broadcast Server

```javascript
const WebSocket = require('ws');

const wss = new WebSocket.Server({
  host: '0.0.0.0',
  port: 8080
});

// Broadcast to all connected clients
function broadcast(data, excludeSocket = null) {
  wss.clients.forEach((client) => {
    if (client !== excludeSocket && client.readyState === WebSocket.OPEN) {
      client.send(data);
    }
  });
}

wss.on('connection', (ws, req) => {
  const ip = req.socket.remoteAddress;
  console.log(`Connected: ${ip} (total: ${wss.clients.size})`);
  
  // Notify all clients of the new connection
  broadcast(JSON.stringify({
    type: 'system',
    message: `New client connected from ${ip}`,
    clients: wss.clients.size
  }));
  
  ws.on('message', (data) => {
    // Re-broadcast received message to everyone except sender
    const message = {
      type: 'message',
      from: ip,
      data: data.toString(),
      timestamp: Date.now()
    };
    
    // Echo to sender
    ws.send(JSON.stringify({ ...message, echo: true }));
    
    // Broadcast to others
    broadcast(JSON.stringify(message), ws);
  });
  
  ws.on('close', () => {
    broadcast(JSON.stringify({
      type: 'system',
      message: `Client ${ip} disconnected`,
      clients: wss.clients.size
    }));
  });
});

console.log('WebSocket server on ws://0.0.0.0:8080');
```

## Room-Based Broadcasting

```javascript
const WebSocket = require('ws');

const wss = new WebSocket.Server({ host: '0.0.0.0', port: 8080 });

// rooms[roomName] = Set<WebSocket>
const rooms = new Map();

function joinRoom(ws, roomName) {
  if (!rooms.has(roomName)) {
    rooms.set(roomName, new Set());
  }
  rooms.get(roomName).add(ws);
  ws.room = roomName;
  console.log(`${ws.clientId} joined room: ${roomName}`);
}

function leaveRoom(ws) {
  if (ws.room && rooms.has(ws.room)) {
    rooms.get(ws.room).delete(ws);
    if (rooms.get(ws.room).size === 0) {
      rooms.delete(ws.room);
    }
    ws.room = null;
  }
}

function broadcastToRoom(roomName, data, excludeWs = null) {
  const room = rooms.get(roomName);
  if (!room) return 0;
  
  let sent = 0;
  room.forEach((client) => {
    if (client !== excludeWs && client.readyState === WebSocket.OPEN) {
      client.send(data);
      sent++;
    }
  });
  return sent;
}

let clientIdCounter = 0;

wss.on('connection', (ws, req) => {
  ws.clientId = `client-${++clientIdCounter}`;
  ws.room = null;
  
  ws.send(JSON.stringify({
    type: 'welcome',
    id: ws.clientId,
    message: 'Connected. Join a room with {"action":"join","room":"roomName"}'
  }));
  
  ws.on('message', (rawData) => {
    let data;
    try {
      data = JSON.parse(rawData.toString());
    } catch {
      ws.send(JSON.stringify({ type: 'error', message: 'Invalid JSON' }));
      return;
    }
    
    switch (data.action) {
      case 'join':
        if (ws.room) leaveRoom(ws);
        joinRoom(ws, data.room);
        ws.send(JSON.stringify({ type: 'joined', room: data.room }));
        broadcastToRoom(data.room, JSON.stringify({
          type: 'system',
          message: `${ws.clientId} joined`,
          room: data.room
        }), ws);
        break;
      
      case 'leave':
        if (ws.room) {
          const room = ws.room;
          leaveRoom(ws);
          ws.send(JSON.stringify({ type: 'left', room }));
        }
        break;
      
      case 'message':
        if (!ws.room) {
          ws.send(JSON.stringify({ type: 'error', message: 'Join a room first' }));
          break;
        }
        
        const messagePayload = JSON.stringify({
          type: 'message',
          from: ws.clientId,
          room: ws.room,
          text: data.text,
          ts: Date.now()
        });
        
        // Echo to sender
        ws.send(messagePayload);
        
        // Broadcast to room
        const count = broadcastToRoom(ws.room, messagePayload, ws);
        console.log(`[${ws.room}] ${ws.clientId}: ${data.text} (sent to ${count} others)`);
        break;
      
      case 'broadcast_all':
        // Broadcast to ALL clients (not just room)
        wss.clients.forEach((client) => {
          if (client.readyState === WebSocket.OPEN) {
            client.send(JSON.stringify({
              type: 'announcement',
              from: ws.clientId,
              text: data.text
            }));
          }
        });
        break;
    }
  });
  
  ws.on('close', () => {
    if (ws.room) {
      broadcastToRoom(ws.room, JSON.stringify({
        type: 'system',
        message: `${ws.clientId} left`
      }));
      leaveRoom(ws);
    }
  });
});

console.log('Room-based WebSocket server on ws://0.0.0.0:8080');
```

## Binary Broadcasting

```javascript
// Broadcast binary data (images, files, audio)
function broadcastBinary(buffer, excludeWs = null) {
  wss.clients.forEach((client) => {
    if (client !== excludeWs && client.readyState === WebSocket.OPEN) {
      client.send(buffer, { binary: true });
    }
  });
}

// Usage: broadcast a typed message with type prefix
function sendTyped(ws, type, payload) {
  // First byte = message type, rest = payload
  const typeBuffer = Buffer.from([type]);
  const combined = Buffer.concat([typeBuffer, Buffer.from(payload)]);
  ws.send(combined, { binary: true });
}
```

## Conclusion

WebSocket broadcasting with Node.js's `ws` library is straightforward - iterate over `wss.clients` and send to each connected client. For production applications with many rooms, use a `Map<roomName, Set<WebSocket>>` data structure for O(room size) broadcasts instead of O(all clients). For multi-server deployments, use Redis pub-sub to broadcast across server instances.
