# How to Scale WebSocket Connections with MongoDB Change Streams

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, WebSocket, Change Streams, Scaling, Node.js

Description: Learn how to scale real-time WebSocket applications backed by MongoDB change streams across multiple server instances using Redis pub/sub.

---

## The Scaling Challenge

A single Node.js server can maintain a MongoDB change stream and broadcast to connected WebSocket clients. But when you scale to multiple server instances (for load balancing or redundancy), each instance has its own change stream and its own set of connected clients.

The problem: a client connected to Server A will not receive events that Server B emits to its local clients - even if they originate from the same MongoDB change.

## Solution: Redis Pub/Sub as a Message Bus

Use Redis to broadcast MongoDB change events across all server instances:

```text
MongoDB Change Stream
        |
    Server 1 ---> Redis PUBLISH "changes"
    Server 2 ---> Redis PUBLISH "changes"
                          |
              Redis SUBSCRIBE "changes"
              /           |           \
         Server 1    Server 2    Server 3
             |           |           |
         Clients     Clients     Clients
```

## Setting Up the Infrastructure

Install dependencies:

```bash
npm install mongodb socket.io express ioredis
```

## Shared Redis Pub/Sub Server

```javascript
// server.js
const express = require("express")
const http = require("http")
const { Server } = require("socket.io")
const { createAdapter } = require("@socket.io/redis-adapter")
const { MongoClient } = require("mongodb")
const Redis = require("ioredis")

const app = express()
const server = http.createServer(app)
const io = new Server(server)

// Create Redis pub/sub clients
const pubClient = new Redis(process.env.REDIS_URL || "redis://localhost:6379")
const subClient = pubClient.duplicate()

// Set Socket.io to use Redis adapter for multi-server broadcasting
io.adapter(createAdapter(pubClient, subClient))

async function startServer() {
  const mongoClient = new MongoClient(process.env.MONGO_URI)
  await mongoClient.connect()
  const db = mongoClient.db("myapp")

  // Only ONE change stream per process is needed
  // Socket.io Redis adapter handles broadcasting to all server instances
  const changeStream = db.collection("orders").watch([], {
    fullDocument: "updateLookup"
  })

  changeStream.on("change", event => {
    // io.emit works across all server instances via Redis adapter
    io.emit("orderChange", {
      type: event.operationType,
      id: event.documentKey._id.toString(),
      document: event.fullDocument
    })
  })

  io.on("connection", socket => {
    console.log(`Client ${socket.id} connected to server ${process.pid}`)
  })

  const PORT = process.env.PORT || 3000
  server.listen(PORT, () => {
    console.log(`Server ${process.pid} listening on port ${PORT}`)
  })
}

startServer().catch(console.error)
```

## Partitioned Change Streams

For high-throughput systems, partition the change stream by collection or filter:

```javascript
// Each server instance watches a subset of changes
const SHARD_ID = parseInt(process.env.SHARD_ID || "0")
const TOTAL_SHARDS = parseInt(process.env.TOTAL_SHARDS || "3")

// Distribute change stream watching across servers by modding the document ID
const changeStream = db.collection("events").watch([
  {
    $match: {
      $expr: {
        $eq: [
          { $mod: [{ $toLong: "$documentKey._id" }, TOTAL_SHARDS] },
          SHARD_ID
        ]
      }
    }
  }
])
```

## Connection Limits and Backpressure

Each change stream consumes a server-side cursor. On a replica set, each `watch()` call opens a cursor on the primary:

```javascript
// Limit the number of change streams per server
const MAX_CHANGE_STREAMS = 10
let activeStreams = 0

function watchCollection(collection, pipeline = []) {
  if (activeStreams >= MAX_CHANGE_STREAMS) {
    throw new Error("Change stream limit reached")
  }

  activeStreams++
  const stream = collection.watch(pipeline, { fullDocument: "updateLookup" })

  stream.on("close", () => {
    activeStreams--
    console.log("Stream closed, active streams:", activeStreams)
  })

  return stream
}
```

## Client-Side WebSocket with Reconnection

```javascript
// Browser client with exponential backoff reconnection
class RealtimeClient {
  constructor(url) {
    this.url = url
    this.retryDelay = 1000
    this.maxRetryDelay = 30000
    this.connect()
  }

  connect() {
    this.ws = new WebSocket(this.url)

    this.ws.onopen = () => {
      console.log("Connected")
      this.retryDelay = 1000  // reset backoff
    }

    this.ws.onmessage = (event) => {
      const data = JSON.parse(event.data)
      this.handleUpdate(data)
    }

    this.ws.onclose = () => {
      console.log(`Disconnected, retrying in ${this.retryDelay}ms`)
      setTimeout(() => {
        this.retryDelay = Math.min(this.retryDelay * 2, this.maxRetryDelay)
        this.connect()
      }, this.retryDelay)
    }
  }

  handleUpdate(data) {
    console.log("Update:", data.type, data.id)
    // update UI
  }
}

const client = new RealtimeClient("wss://your-server.example.com")
```

## Horizontal Scaling with PM2

```javascript
// ecosystem.config.js for PM2 cluster mode
module.exports = {
  apps: [{
    name: "realtime-server",
    script: "server.js",
    instances: "max",  // use all CPU cores
    exec_mode: "cluster",
    env: {
      MONGO_URI: "mongodb+srv://...",
      REDIS_URL: "redis://redis:6379",
      NODE_ENV: "production"
    }
  }]
}
```

```bash
pm2 start ecosystem.config.js
pm2 status
```

## Monitoring WebSocket Connections

```javascript
// Track connection metrics
setInterval(() => {
  const sockets = io.sockets.sockets.size
  console.log(`Active WebSocket connections: ${sockets}`)

  // Report to monitoring
  process.emit("metrics", { websocketConnections: sockets })
}, 30000)
```

## Summary

Scale MongoDB change stream applications across multiple servers by using the Socket.io Redis adapter to fan out events from any server instance to all connected clients. Each server instance maintains a change stream and publishes events through Redis pub/sub, ensuring all WebSocket clients receive updates regardless of which server they are connected to. For very high throughput, partition change streams by document ID across server instances.
