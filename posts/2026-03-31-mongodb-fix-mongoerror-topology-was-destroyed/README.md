# How to Fix MongoError: Topology Was Destroyed in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Topology, Connection Pool, Error, Troubleshooting

Description: Learn why MongoError Topology Was Destroyed occurs and how to fix it by managing client lifecycle, connection pools, and graceful shutdown properly.

---

## Understanding the Error

`MongoError: Topology was destroyed` means an operation was attempted on a `MongoClient` that has already been closed or that lost its connection and was garbage-collected. The underlying topology (the driver's internal connection manager) no longer exists.

```text
MongoError: Topology was destroyed
    at Server.command (node_modules/mongodb/lib/core/topologies/server.js)
```

## Common Causes

### 1. Calling `client.close()` Too Early

The most common cause is closing the client while queries are still in flight, or reusing a closed client:

```javascript
// BAD - client closed before query resolves
const client = new MongoClient(uri);
await client.connect();
const db = client.db('mydb');
client.close(); // closes immediately
const docs = await db.collection('users').find({}).toArray(); // topology destroyed
```

**Fix:** await all operations before closing, or restructure to keep the client alive for the application's lifetime:

```javascript
// GOOD - keep client alive as a module-level singleton
let client;

export async function getDb() {
  if (!client) {
    client = new MongoClient(process.env.MONGODB_URI);
    await client.connect();
  }
  return client.db('mydb');
}

// Only close on process exit
process.on('SIGTERM', async () => {
  if (client) await client.close();
  process.exit(0);
});
```

### 2. Creating a New Client Per Request

Opening and closing a `MongoClient` per HTTP request is expensive and prone to race conditions. The driver has a built-in connection pool - reuse one client:

```javascript
// BAD - new client per request
app.get('/users', async (req, res) => {
  const client = new MongoClient(uri);
  await client.connect();
  const users = await client.db('mydb').collection('users').find({}).toArray();
  await client.close();
  res.json(users);
});

// GOOD - module-level singleton
import { getDb } from './db';

app.get('/users', async (req, res) => {
  const db = await getDb();
  const users = await db.collection('users').find({}).toArray();
  res.json(users);
});
```

### 3. Serverless or Lambda Environments

In serverless functions, the execution context may freeze and reuse the client from a previous invocation, which could have gone stale. Implement a reconnect guard:

```javascript
let client = null;

export async function getClient() {
  if (!client || !client.topology || !client.topology.isConnected()) {
    client = new MongoClient(process.env.MONGODB_URI);
    await client.connect();
  }
  return client;
}
```

### 4. Uncaught Errors Destroying the Client

If the driver encounters an unrecoverable network error, it may tear down the topology. Wrap operations with try/catch and reconnect on fatal errors:

```javascript
async function withRetry(fn, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      return await fn();
    } catch (err) {
      if (err.message.includes('Topology was destroyed') && i < retries - 1) {
        client = null; // force reconnect on next call
        continue;
      }
      throw err;
    }
  }
}
```

## Diagnosing With Monitoring

Enable topology monitoring events to trace what destroyed the topology:

```javascript
client.on('topologyClosed', (event) => {
  console.error('Topology closed:', event);
});
```

## Summary

`Topology was destroyed` almost always traces back to premature client closure or creating a new client per request. Use a module-level singleton `MongoClient`, avoid calling `close()` during request handling, and implement graceful shutdown hooks. For serverless environments, add a connectivity check before reusing a cached client.
