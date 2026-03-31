# How to Build Real-Time Updates with MongoDB and Socket.io

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Socket.io, Real-Time, Change Streams, Node.js

Description: Learn how to combine MongoDB change streams with Socket.io to push real-time database updates to browser clients as they happen.

---

## Architecture Overview

MongoDB change streams watch for database changes and emit events. Socket.io broadcasts those events to connected browser clients. Together, they enable real-time dashboards, live feeds, and collaborative applications.

```text
MongoDB Atlas/Replica Set
         |
    Change Stream
         |
    Node.js Server (Express + Socket.io)
         |
    WebSocket (Socket.io)
         |
    Browser Clients
```

## Server Setup

Install dependencies:

```bash
npm install mongodb socket.io express
```

## Basic Server with Change Stream and Socket.io

```javascript
// server.js
const express = require("express")
const http = require("http")
const { Server } = require("socket.io")
const { MongoClient } = require("mongodb")

const app = express()
const server = http.createServer(app)
const io = new Server(server, {
  cors: { origin: "*" }
})

const MONGO_URI = process.env.MONGO_URI || "mongodb://localhost:27017"
const DB_NAME = "myapp"

async function startServer() {
  const client = new MongoClient(MONGO_URI)
  await client.connect()
  const db = client.db(DB_NAME)

  // Watch for changes in the orders collection
  const changeStream = db.collection("orders").watch([], {
    fullDocument: "updateLookup"  // include full document on updates
  })

  changeStream.on("change", event => {
    console.log("Change detected:", event.operationType)

    // Broadcast to all connected clients
    io.emit("orderChange", {
      type: event.operationType,
      documentId: event.documentKey._id,
      document: event.fullDocument,
      updatedFields: event.updateDescription?.updatedFields
    })
  })

  changeStream.on("error", err => {
    console.error("Change stream error:", err)
  })

  // Handle Socket.io connections
  io.on("connection", socket => {
    console.log("Client connected:", socket.id)

    socket.on("disconnect", () => {
      console.log("Client disconnected:", socket.id)
    })
  })

  server.listen(3000, () => {
    console.log("Server running on http://localhost:3000")
  })
}

startServer().catch(console.error)
```

## Filtering Change Stream Events

Watch only specific operations or documents:

```javascript
// Only watch for inserts and updates to pending orders
const pipeline = [
  {
    $match: {
      $or: [
        { operationType: "insert" },
        {
          operationType: "update",
          "fullDocument.status": { $in: ["pending", "processing"] }
        }
      ]
    }
  }
]

const changeStream = db.collection("orders").watch(pipeline, {
  fullDocument: "updateLookup"
})
```

## Client-Side Browser Code

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
  <title>Real-Time Orders</title>
  <script src="/socket.io/socket.io.js"></script>
</head>
<body>
  <h1>Live Orders</h1>
  <div id="orders"></div>

  <script>
    const socket = io()

    socket.on("connect", () => {
      console.log("Connected to server")
    })

    socket.on("orderChange", (data) => {
      console.log("Order update:", data)

      const div = document.getElementById("orders")
      const entry = document.createElement("p")
      entry.textContent = `${data.type.toUpperCase()}: Order ${data.documentId}`
      div.prepend(entry)

      // Update UI based on operation type
      if (data.type === "insert") {
        addOrderToUI(data.document)
      } else if (data.type === "update") {
        updateOrderInUI(data.documentId, data.updatedFields)
      } else if (data.type === "delete") {
        removeOrderFromUI(data.documentId)
      }
    })

    function addOrderToUI(order) {
      // Add order to the list
    }
    function updateOrderInUI(id, fields) {
      // Find and update the order card
    }
    function removeOrderFromUI(id) {
      // Remove order from the list
    }
  </script>
</body>
</html>
```

## Room-Based Broadcasting

Send updates only to relevant clients using Socket.io rooms:

```javascript
// Server: broadcast to users watching a specific dashboard
io.on("connection", socket => {
  socket.on("watchDashboard", (dashboardId) => {
    socket.join(`dashboard:${dashboardId}`)
    console.log(`Socket ${socket.id} joined dashboard:${dashboardId}`)
  })

  socket.on("leaveDashboard", (dashboardId) => {
    socket.leave(`dashboard:${dashboardId}`)
  })
})

// Only emit to clients watching the relevant dashboard
changeStream.on("change", event => {
  if (event.fullDocument) {
    const dashboardId = event.fullDocument.dashboardId
    io.to(`dashboard:${dashboardId}`).emit("dataUpdate", {
      type: event.operationType,
      data: event.fullDocument
    })
  }
})
```

## Resuming Change Streams After Reconnect

Handle server restarts by resuming from the last processed event:

```javascript
let resumeToken = null

function watchWithResume(collection) {
  const options = resumeToken
    ? { fullDocument: "updateLookup", resumeAfter: resumeToken }
    : { fullDocument: "updateLookup" }

  const changeStream = collection.watch([], options)

  changeStream.on("change", event => {
    resumeToken = event._id  // save for resume
    io.emit("change", event)
  })

  changeStream.on("error", async err => {
    console.error("Stream error, reconnecting...", err)
    await new Promise(r => setTimeout(r, 1000))
    watchWithResume(collection)
  })

  return changeStream
}
```

## Summary

Combine MongoDB change streams with Socket.io to deliver real-time database updates to browser clients. Watch the change stream on the server, transform events as needed, and emit them via Socket.io to connected clients. Use pipeline filters to reduce unnecessary events, Socket.io rooms for targeted broadcasts, and resume tokens to survive server restarts without missing events.
