# How to Use Connection Pooling Effectively in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Connection Pooling, Performance, Node.js, Driver Configuration

Description: Learn how MongoDB connection pooling works, how to configure pool size and timeouts, and best practices for reusing connections across your application.

---

## Overview

MongoDB drivers maintain a pool of persistent TCP connections to the server. Reusing connections from the pool avoids the overhead of establishing new connections for each operation. Proper pool configuration is critical for high-throughput applications.

## How Connection Pooling Works

When your application starts, the driver creates a pool with a minimum number of connections. As load increases, it opens additional connections up to the configured maximum. When a connection is no longer needed, it is returned to the pool rather than closed.

```text
Application Requests
       |
  Driver Pool (min: 5, max: 100)
  [ conn1 ][ conn2 ][ conn3 ][ idle... ]
       |
  MongoDB Server
```

## Configuring the Pool in Node.js

```javascript
const { MongoClient } = require("mongodb");

const client = new MongoClient("mongodb://localhost:27017", {
  maxPoolSize: 50,          // max connections in pool (default: 100)
  minPoolSize: 5,           // connections kept open when idle (default: 0)
  maxIdleTimeMS: 30000,     // close idle connections after 30s
  waitQueueTimeoutMS: 5000, // throw error if no connection available in 5s
  connectTimeoutMS: 10000,  // timeout for establishing a new connection
  socketTimeoutMS: 45000    // timeout for individual socket operations
});

// IMPORTANT: Create the client once and reuse it
await client.connect();

// Export and reuse this client throughout your app
module.exports = client;
```

## Reusing the Client - The Critical Rule

The most common connection pooling mistake is creating a new `MongoClient` for every request:

```javascript
// BAD: Creates a new connection pool per request
app.get("/users", async (req, res) => {
  const client = new MongoClient(uri);
  await client.connect();
  const users = await client.db("myapp").collection("users").find().toArray();
  await client.close();   // closes all pooled connections
  res.json(users);
});

// GOOD: Reuse the client singleton
const client = new MongoClient(uri, { maxPoolSize: 50 });
await client.connect();

app.get("/users", async (req, res) => {
  const users = await client.db("myapp").collection("users").find().toArray();
  res.json(users);
});
```

## Singleton Pattern for Express.js

```javascript
// db.js
const { MongoClient } = require("mongodb");

let client;

async function getClient() {
  if (!client) {
    client = new MongoClient(process.env.MONGODB_URI, {
      maxPoolSize: 100,
      minPoolSize: 10,
      maxIdleTimeMS: 60000,
      waitQueueTimeoutMS: 10000
    });
    await client.connect();
  }
  return client;
}

module.exports = { getClient };

// In your route handlers
const { getClient } = require("./db");

app.get("/orders", async (req, res) => {
  const client = await getClient();
  const orders = await client.db("store").collection("orders")
    .find({ status: "pending" })
    .limit(50)
    .toArray();
  res.json(orders);
});
```

## Monitoring Pool Usage

Use the driver's CMAP (Connection Monitoring and Pooling) events:

```javascript
client.on("connectionPoolCreated", (event) => {
  console.log("Pool created:", event.address);
});

client.on("connectionCheckedOut", (event) => {
  console.log("Connection checked out, pool ID:", event.connectionId);
});

client.on("connectionPoolCleared", (event) => {
  console.log("Pool cleared:", event.address);
});
```

## Configuring Pool Size

Choose `maxPoolSize` based on your expected concurrency:

- For a web server handling 50 concurrent requests: set `maxPoolSize` to 50-100.
- For a microservice with low concurrency: 10-20 is often sufficient.
- MongoDB can handle thousands of connections, but each connection consumes ~1 MB of RAM on the server.

```javascript
// Rule of thumb: maxPoolSize = peak concurrent operations * 1.5
const maxConcurrentRequests = 100;
const client = new MongoClient(uri, {
  maxPoolSize: Math.ceil(maxConcurrentRequests * 1.5)
});
```

## PyMongo Connection Pool Configuration

```python
from pymongo import MongoClient

client = MongoClient(
    "mongodb://localhost:27017/",
    maxPoolSize=50,
    minPoolSize=5,
    maxIdleTimeMS=30000,
    waitQueueTimeoutMS=5000
)

# Reuse this client for the lifetime of the application
db = client["myapp"]
```

## Summary

MongoDB connection pooling is managed automatically by the driver, but you must configure it correctly and reuse the `MongoClient` singleton throughout your application. Set `maxPoolSize` proportional to your expected concurrency, use `minPoolSize` to keep warm connections ready, and set `waitQueueTimeoutMS` to surface connection exhaustion quickly. Never create a new client per request - that pattern defeats the pool entirely.
