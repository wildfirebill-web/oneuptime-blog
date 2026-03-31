# How to Build Real-Time Updates with MongoDB and ws Library

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, WebSocket, Real-Time, Node.js, Change Stream

Description: Build a real-time data pipeline using MongoDB change streams and the lightweight ws library to stream database updates to browser clients.

---

## Overview

The `ws` library is the most popular bare-bones WebSocket implementation for Node.js. It has zero dependencies and integrates cleanly with MongoDB change streams, making it ideal for lightweight real-time applications that do not need the abstractions provided by Socket.io.

## Project Setup

```bash
mkdir mongo-ws-demo && cd mongo-ws-demo
npm init -y
npm install ws mongodb dotenv
```

Create a `.env` file.

```text
MONGO_URI=mongodb://localhost:27017
DB_NAME=inventory
COLLECTION_NAME=products
WS_PORT=3001
```

## Building the Server

```javascript
require('dotenv').config();
const { MongoClient } = require('mongodb');
const { WebSocketServer, OPEN } = require('ws');

const { MONGO_URI, DB_NAME, COLLECTION_NAME, WS_PORT } = process.env;

// Track clients with their subscription filters
const clients = new Set();

async function startServer() {
  const client = new MongoClient(MONGO_URI);
  await client.connect();
  console.log('Connected to MongoDB');

  const col = client.db(DB_NAME).collection(COLLECTION_NAME);
  const wss = new WebSocketServer({ port: Number(WS_PORT) });

  wss.on('connection', (ws, req) => {
    clients.add(ws);
    console.log(`Client connected. Total: ${clients.size}`);

    ws.on('close', () => {
      clients.delete(ws);
      console.log(`Client disconnected. Total: ${clients.size}`);
    });

    ws.on('error', (err) => console.error('WebSocket error:', err.message));

    // Send a welcome message
    ws.send(JSON.stringify({ type: 'connected', message: 'Listening for changes' }));
  });

  // Single change stream for the collection
  const pipeline = [
    { $match: { operationType: { $in: ['insert', 'update', 'delete', 'replace'] } } }
  ];

  const changeStream = col.watch(pipeline, { fullDocument: 'updateLookup' });

  changeStream.on('change', (event) => {
    const message = JSON.stringify({
      type: event.operationType,
      documentKey: event.documentKey,
      fullDocument: event.fullDocument ?? null,
      updateDescription: event.updateDescription ?? null,
      timestamp: new Date().toISOString()
    });

    clients.forEach((ws) => {
      if (ws.readyState === OPEN) ws.send(message);
    });
  });

  changeStream.on('error', (err) => {
    console.error('Change stream error:', err.message);
  });

  console.log(`WebSocket server running on ws://localhost:${WS_PORT}`);
}

startServer().catch(console.error);
```

## Browser Client

```html
<!DOCTYPE html>
<html lang="en">
<head><meta charset="UTF-8"><title>Live Inventory</title></head>
<body>
  <ul id="log"></ul>
  <script>
    const ws = new WebSocket('ws://localhost:3001');
    const log = document.getElementById('log');

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      const li = document.createElement('li');
      li.textContent = `[${data.type}] ${JSON.stringify(data.documentKey)} at ${data.timestamp}`;
      log.prepend(li);
    };

    ws.onerror = (e) => console.error('WebSocket error', e);
    ws.onclose = () => console.log('Connection closed');
  </script>
</body>
</html>
```

## Broadcasting Only Relevant Fields

Avoid sending entire documents when clients need only specific fields.

```javascript
changeStream.on('change', (event) => {
  if (event.operationType === 'update') {
    const message = JSON.stringify({
      type: 'update',
      id: event.documentKey._id,
      changes: event.updateDescription.updatedFields
    });
    clients.forEach((ws) => {
      if (ws.readyState === OPEN) ws.send(message);
    });
  }
});
```

## Testing with wscat

```bash
npx wscat -c ws://localhost:3001
```

Insert a document in another terminal to trigger a live event.

```bash
mongosh --eval 'db.products.insertOne({ name: "Laptop", qty: 50, price: 999 })' inventory
```

## Summary

The `ws` library provides a minimal, high-performance WebSocket layer that pairs naturally with MongoDB change streams. Keep a single change stream open per collection, broadcast events to the client set, and filter both in the aggregation pipeline and in the broadcast handler to minimize payload size and network overhead.
