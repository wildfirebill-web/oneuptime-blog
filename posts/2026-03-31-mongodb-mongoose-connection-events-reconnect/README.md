# How to Handle Mongoose Connection Events and Reconnection

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoose, Connection, Reconnection, Node.js

Description: Learn how to listen to Mongoose connection events, handle disconnects gracefully, and configure automatic reconnection for production Node.js apps.

---

## Overview

Mongoose wraps the MongoDB Node.js driver and emits connection lifecycle events. Monitoring these events and configuring reconnection options ensures your application stays resilient during network interruptions or MongoDB restarts.

## Connecting to MongoDB

```javascript
const mongoose = require('mongoose');

async function connectDB() {
  await mongoose.connect(process.env.MONGO_URI, {
    serverSelectionTimeoutMS: 5000,
    socketTimeoutMS: 45000,
    heartbeatFrequencyMS: 10000
  });
}
```

## Listening to Connection Events

Mongoose's connection emits the following events:

```javascript
const db = mongoose.connection;

db.on('connected', () => {
  console.log('Mongoose connected to MongoDB');
});

db.on('error', (err) => {
  console.error('Mongoose connection error:', err.message);
});

db.on('disconnected', () => {
  console.warn('Mongoose disconnected from MongoDB');
});

db.on('reconnected', () => {
  console.log('Mongoose reconnected to MongoDB');
});

db.on('close', () => {
  console.log('Mongoose connection closed');
});
```

## Auto-Reconnection with Retry Loop

Mongoose does not automatically reconnect after an initial connection failure. Implement a retry loop at startup:

```javascript
async function connectWithRetry(uri, maxRetries = 10, delayMs = 3000) {
  for (let i = 1; i <= maxRetries; i++) {
    try {
      await mongoose.connect(uri);
      console.log('Connected to MongoDB');
      return;
    } catch (err) {
      console.warn(`Connection attempt ${i}/${maxRetries} failed:`, err.message);
      if (i < maxRetries) {
        await new Promise(r => setTimeout(r, delayMs));
      } else {
        throw new Error('Could not connect to MongoDB');
      }
    }
  }
}
```

## Handling Disconnection During Runtime

The driver will attempt to reconnect automatically during runtime disconnections. Configure the behavior:

```javascript
await mongoose.connect(uri, {
  serverSelectionTimeoutMS: 30000,  // how long to wait for a server
  heartbeatFrequencyMS: 5000,       // check server health every 5s
  maxPoolSize: 10
});
```

## Checking Connection State

```javascript
const states = {
  0: 'disconnected',
  1: 'connected',
  2: 'connecting',
  3: 'disconnecting'
};

console.log('Current state:', states[mongoose.connection.readyState]);
```

## Graceful Shutdown

Close the connection cleanly on process exit:

```javascript
async function gracefulShutdown(signal) {
  console.log(`Received ${signal}. Closing MongoDB connection...`);
  await mongoose.connection.close();
  console.log('MongoDB connection closed.');
  process.exit(0);
}

process.on('SIGINT', () => gracefulShutdown('SIGINT'));
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
```

## Summary

Mongoose exposes connection lifecycle events - `connected`, `disconnected`, `error`, `reconnected` - on `mongoose.connection`. Implement a startup retry loop to handle initial connection failures, configure `serverSelectionTimeoutMS` and `heartbeatFrequencyMS` for mid-runtime resilience, and always close the connection gracefully on process termination.
