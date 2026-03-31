# How to Build a WebSocket Bridge for Redis Pub/Sub

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Pub/Sub, WebSocket, Real-Time, Node.js, Bridge, Event-Driven

Description: Build a WebSocket bridge that forwards Redis Pub/Sub messages to browser clients in real time, enabling push notifications and live data feeds without polling.

---

Redis Pub/Sub is an efficient server-side messaging primitive, but browsers cannot connect to Redis directly. A WebSocket bridge sits between Redis and the browser: it subscribes to Redis channels and forwards messages to connected WebSocket clients, enabling real-time push updates for dashboards, notifications, and live feeds.

## Architecture

```text
Publishers                Redis                 WS Bridge             Browsers
(services)  ──PUBLISH──► Pub/Sub  ──message──► Node.js  ──ws──►  Dashboard
                          Server                server              Clients
```

## Node.js WebSocket Bridge (ws + ioredis)

```javascript
const WebSocket = require('ws');
const Redis = require('ioredis');

const PORT = 8080;
const REDIS_URL = process.env.REDIS_URL || 'redis://localhost:6379';

const wss = new WebSocket.Server({ port: PORT });
const subscriber = new Redis(REDIS_URL);
const publisher = new Redis(REDIS_URL);  // separate connection for commands

// Map: channel -> Set of WebSocket clients
const channelClients = new Map();

subscriber.on('message', (channel, message) => {
  const clients = channelClients.get(channel);
  if (!clients) return;

  const payload = JSON.stringify({ channel, data: JSON.parse(message) });
  for (const ws of clients) {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(payload);
    }
  }
});

wss.on('connection', (ws, req) => {
  console.log(`Client connected from ${req.socket.remoteAddress}`);
  const subscribed = new Set();

  ws.on('message', async (raw) => {
    let msg;
    try { msg = JSON.parse(raw); } catch { return; }

    if (msg.action === 'subscribe' && msg.channel) {
      const ch = msg.channel;
      if (!channelClients.has(ch)) {
        channelClients.set(ch, new Set());
        await subscriber.subscribe(ch);
        console.log(`Redis: subscribed to ${ch}`);
      }
      channelClients.get(ch).add(ws);
      subscribed.add(ch);
      ws.send(JSON.stringify({ type: 'subscribed', channel: ch }));
    }

    if (msg.action === 'unsubscribe' && msg.channel) {
      unsubscribeClient(ws, msg.channel);
    }
  });

  ws.on('close', () => {
    for (const ch of subscribed) {
      unsubscribeClient(ws, ch);
    }
    console.log('Client disconnected');
  });
});

function unsubscribeClient(ws, channel) {
  const clients = channelClients.get(channel);
  if (!clients) return;
  clients.delete(ws);
  if (clients.size === 0) {
    channelClients.delete(channel);
    subscriber.unsubscribe(channel);
    console.log(`Redis: unsubscribed from ${channel}`);
  }
}

console.log(`WebSocket bridge running on ws://localhost:${PORT}`);
```

## Browser Client (JavaScript)

```javascript
const ws = new WebSocket('ws://localhost:8080');

ws.onopen = () => {
  // Subscribe to a Redis channel via the bridge
  ws.send(JSON.stringify({ action: 'subscribe', channel: 'events:orders' }));
  ws.send(JSON.stringify({ action: 'subscribe', channel: 'notifications:user:42' }));
};

ws.onmessage = (event) => {
  const { channel, data } = JSON.parse(event.data);
  console.log(`[${channel}]`, data);

  if (channel === 'events:orders') {
    updateOrdersUI(data);
  }
};

ws.onerror = (err) => console.error('WebSocket error:', err);
ws.onclose = () => console.log('Disconnected from bridge');
```

## Publisher (Any Service)

```python
import redis
import json
import time

r = redis.Redis(host='localhost', port=6379)

# Publish an order event; the bridge forwards it to all WebSocket subscribers
r.publish('events:orders', json.dumps({
    'order_id': 1001,
    'user_id': 42,
    'status': 'paid',
    'total': 99.99,
    'timestamp': time.time(),
}))
```

## Add Authentication with JWT

```javascript
const jwt = require('jsonwebtoken');
const SECRET = process.env.JWT_SECRET;

wss.on('connection', (ws, req) => {
  const token = new URL(req.url, 'http://x').searchParams.get('token');
  let user;
  try {
    user = jwt.verify(token, SECRET);
  } catch {
    ws.close(4001, 'Unauthorized');
    return;
  }

  // Restrict subscriptions to user's own channels
  ws.on('message', (raw) => {
    const msg = JSON.parse(raw);
    if (msg.action === 'subscribe') {
      // Enforce channel access policy
      if (!msg.channel.startsWith(`notifications:user:${user.id}`)) {
        ws.send(JSON.stringify({ type: 'error', message: 'Access denied' }));
        return;
      }
      // ... proceed with subscription
    }
  });
});
```

## Monitor Bridge Health

```javascript
setInterval(() => {
  const totalClients = wss.clients.size;
  const totalChannels = channelClients.size;
  console.log(`Clients: ${totalClients}, Channels: ${totalChannels}`);
}, 10000);
```

## Summary

A WebSocket bridge connecting Redis Pub/Sub to browsers requires only a small Node.js server using `ws` and `ioredis`. The key design points are: maintain a map of channel-to-client sets, subscribe to Redis channels on demand (and unsubscribe when the last client leaves), and enforce access control at the WebSocket handshake layer. This pattern powers real-time dashboards, notification systems, and live data feeds with minimal infrastructure.
