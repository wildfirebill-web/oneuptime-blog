# How to Use MongoDB Change Streams with WebSocket Servers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Change Stream, WebSocket, Real-Time, Node.js

Description: Learn how to connect MongoDB change streams directly to a native WebSocket server to push real-time database updates to connected clients.

---

## Overview

MongoDB change streams let you react to data changes at the collection, database, or deployment level. Pairing them with a native WebSocket server gives you a lightweight way to push updates to browsers and other clients without polling.

## Setting Up the WebSocket Server

Install the `ws` package alongside the MongoDB Node.js driver.

```bash
npm install ws mongodb
```

Create the server entry point.

```javascript
const { MongoClient } = require('mongodb');
const { WebSocketServer } = require('ws');

const MONGO_URI = process.env.MONGO_URI || 'mongodb://localhost:27017';
const DB_NAME = 'shop';
const COLLECTION = 'orders';

async function main() {
  const client = new MongoClient(MONGO_URI);
  await client.connect();

  const collection = client.db(DB_NAME).collection(COLLECTION);
  const wss = new WebSocketServer({ port: 8080 });

  console.log('WebSocket server listening on port 8080');

  // Open one change stream shared across all WebSocket clients
  const changeStream = collection.watch([], { fullDocument: 'updateLookup' });

  changeStream.on('change', (event) => {
    const payload = JSON.stringify(event);
    wss.clients.forEach((ws) => {
      if (ws.readyState === ws.OPEN) {
        ws.send(payload);
      }
    });
  });

  wss.on('connection', (ws) => {
    console.log('Client connected');
    ws.on('close', () => console.log('Client disconnected'));
  });

  process.on('SIGINT', async () => {
    await changeStream.close();
    await client.close();
    process.exit(0);
  });
}

main().catch(console.error);
```

## Filtering the Change Stream

Avoid sending unnecessary noise to clients by filtering with an aggregation pipeline.

```javascript
const pipeline = [
  { $match: { operationType: { $in: ['insert', 'update', 'replace'] } } },
  { $match: { 'fullDocument.status': { $in: ['pending', 'shipped'] } } }
];

const changeStream = collection.watch(pipeline, { fullDocument: 'updateLookup' });
```

## Handling Errors and Reconnection

Change streams can be interrupted if the replica set has issues. Wrap in a retry loop.

```javascript
async function watchWithRetry(collection, wss) {
  let resumeToken = null;

  while (true) {
    try {
      const opts = resumeToken
        ? { fullDocument: 'updateLookup', resumeAfter: resumeToken }
        : { fullDocument: 'updateLookup' };

      const changeStream = collection.watch([], opts);

      for await (const event of changeStream) {
        resumeToken = event._id;
        const payload = JSON.stringify(event);
        wss.clients.forEach((ws) => {
          if (ws.readyState === ws.OPEN) ws.send(payload);
        });
      }
    } catch (err) {
      console.error('Change stream error, retrying in 2s:', err.message);
      await new Promise((r) => setTimeout(r, 2000));
    }
  }
}
```

## Testing the Integration

Use the `wscat` command-line tool to listen for events.

```bash
npx wscat -c ws://localhost:8080
```

Then insert a document from the Mongo shell to trigger an event.

```javascript
db.orders.insertOne({ item: 'widget', qty: 10, status: 'pending' });
```

The connected WebSocket client should immediately receive the change event JSON.

## Performance Considerations

- Share a single change stream across all WebSocket clients rather than opening one per client. Each change stream consumes an oplog cursor on the server.
- Use a pipeline to pre-filter events so that only matching changes traverse the network.
- Run the WebSocket server as a separate process and scale it horizontally behind a load balancer. Use a shared `resumeToken` store (Redis or another MongoDB collection) so any instance can resume the stream.

## Summary

MongoDB change streams combined with a native WebSocket server provide a simple, dependency-light approach to real-time updates. Open one shared change stream, broadcast events to all connected clients, store the resume token for fault tolerance, and use aggregation pipelines to filter irrelevant events before they reach your application layer.
