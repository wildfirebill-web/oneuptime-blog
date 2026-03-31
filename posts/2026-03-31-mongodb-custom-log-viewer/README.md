# How to Build a Custom Log Viewer with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Logging, Node.js, Express, Dashboard

Description: Learn how to build a custom web-based log viewer backed by MongoDB with real-time streaming, filtering, and search using Node.js, Express, and Server-Sent Events.

---

## Overview

A custom log viewer gives you full control over how your logs are displayed, filtered, and searched. By combining MongoDB's aggregation pipeline with a Node.js Express API and Server-Sent Events (SSE), you can build a real-time log dashboard without any third-party log management tool.

## Project Structure

```text
log-viewer/
  server.js
  public/
    index.html
    app.js
```

## Setting Up the Express API

```javascript
const express = require('express');
const { MongoClient } = require('mongodb');

const app = express();
app.use(express.static('public'));

const client = new MongoClient(process.env.MONGODB_URI);
const db = client.db('logs');
const collection = db.collection('app_logs');

// Query logs with filters and pagination
app.get('/api/logs', async (req, res) => {
  const { level, service, search, limit = 100, before } = req.query;
  const query = {};

  if (level) query.level = level;
  if (service) query.service = service;
  if (search) query.$text = { $search: search };
  if (before) query.timestamp = { $lt: new Date(before) };

  const logs = await collection
    .find(query)
    .sort({ timestamp: -1 })
    .limit(parseInt(limit))
    .toArray();

  res.json(logs);
});

client.connect().then(() => app.listen(3000, () => console.log('Log viewer running on port 3000')));
```

## Real-Time Log Streaming with Server-Sent Events

Use a tailable cursor on a capped collection to stream new logs to the browser in real time:

```javascript
app.get('/api/logs/stream', async (req, res) => {
  res.set({
    'Content-Type': 'text/event-stream',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
  });
  res.flushHeaders();

  // Start from the latest document
  const latest = await collection.findOne({}, { sort: { $natural: -1 } });
  const query = latest ? { _id: { $gt: latest._id } } : {};

  const cursor = collection.find(query, {
    tailable: true,
    awaitData: true,
  });

  req.on('close', () => cursor.close());

  for await (const doc of cursor) {
    res.write(`data: ${JSON.stringify(doc)}\n\n`);
  }
});
```

## Frontend - Log Viewer HTML

```html
<!DOCTYPE html>
<html>
<head>
  <title>Log Viewer</title>
  <style>
    body { font-family: monospace; background: #1a1a1a; color: #e0e0e0; padding: 20px; }
    .error { color: #ff6b6b; }
    .warn  { color: #ffd93d; }
    .info  { color: #6bcb77; }
    .debug { color: #888; }
    #logs  { max-height: 80vh; overflow-y: auto; }
  </style>
</head>
<body>
  <input id="levelFilter" placeholder="Filter by level..." />
  <input id="serviceFilter" placeholder="Filter by service..." />
  <button onclick="loadLogs()">Search</button>
  <div id="logs"></div>
  <script src="app.js"></script>
</body>
</html>
```

## Frontend - JavaScript Client

```javascript
const logsDiv = document.getElementById('logs');

function renderLog(log) {
  const line = document.createElement('div');
  line.className = log.level;
  line.textContent = `[${new Date(log.timestamp).toISOString()}] [${log.level.toUpperCase()}] ${log.service}: ${log.message}`;
  logsDiv.prepend(line);
}

async function loadLogs() {
  const level = document.getElementById('levelFilter').value;
  const service = document.getElementById('serviceFilter').value;
  const params = new URLSearchParams({ limit: 200 });
  if (level) params.set('level', level);
  if (service) params.set('service', service);

  const response = await fetch(`/api/logs?${params}`);
  const logs = await response.json();
  logsDiv.innerHTML = '';
  logs.forEach(renderLog);
}

// Stream real-time logs
const stream = new EventSource('/api/logs/stream');
stream.onmessage = (event) => renderLog(JSON.parse(event.data));

loadLogs();
```

## Adding Log Level Counts for the Dashboard

```javascript
app.get('/api/logs/summary', async (req, res) => {
  const since = new Date(Date.now() - 3600 * 1000);
  const summary = await collection.aggregate([
    { $match: { timestamp: { $gte: since } } },
    { $group: { _id: '$level', count: { $sum: 1 } } },
  ]).toArray();
  res.json(summary);
});
```

## Best Practices

- Use a capped collection as the data source for the real-time stream to get tailable cursor support.
- Add a text index on `message` to enable full-text search without scanning all documents.
- Implement cursor pagination using `_id` or `timestamp` (the `before` parameter) rather than `skip` for efficient deep pagination.

## Summary

A MongoDB-backed custom log viewer combines a capped collection for real-time streaming, aggregation queries for filtering and search, and Server-Sent Events for pushing updates to the browser without WebSocket complexity.
