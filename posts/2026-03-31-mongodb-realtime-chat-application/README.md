# How to Build a Real-Time Chat Application with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Chat, Real-Time, Socket.IO, Change Stream

Description: Learn how to build a real-time chat application using MongoDB for message storage, change streams for live delivery, and Socket.IO for WebSocket communication.

---

## Overview

A real-time chat application requires persistent message storage and instant delivery. MongoDB handles message persistence naturally with its document model, and change streams enable real-time event delivery without polling. Combined with Socket.IO, you can build a scalable chat system.

## Schema Design

```javascript
// conversations collection
{
  _id: ObjectId(),
  participants: ["user-1", "user-2"],
  type: "direct",  // or "group"
  name: null,  // group chat name
  createdAt: new Date(),
  lastMessage: { text: "Hey!", sentAt: new Date() }
}

// messages collection
{
  _id: ObjectId(),
  conversationId: ObjectId("..."),
  senderId: "user-1",
  text: "Hey there!",
  readBy: ["user-1"],
  sentAt: new Date()
}
```

## Setting Up Indexes

```javascript
db.messages.createIndex({ conversationId: 1, sentAt: -1 })
db.conversations.createIndex({ participants: 1 })
```

## Express and Socket.IO Server

```javascript
const express = require("express");
const http = require("http");
const { Server } = require("socket.io");
const { MongoClient, ObjectId } = require("mongodb");

const app = express();
const server = http.createServer(app);
const io = new Server(server, { cors: { origin: "*" } });

const mongo = new MongoClient("mongodb://localhost:27017");
let db;

mongo.connect().then(() => {
  db = mongo.db("chatapp");
  console.log("Connected to MongoDB");
});

io.on("connection", (socket) => {
  console.log(`User connected: ${socket.id}`);

  socket.on("join_conversation", (conversationId) => {
    socket.join(conversationId);
  });

  socket.on("send_message", async (data) => {
    const { conversationId, senderId, text } = data;
    const message = {
      conversationId: new ObjectId(conversationId),
      senderId,
      text,
      readBy: [senderId],
      sentAt: new Date()
    };

    const result = await db.collection("messages").insertOne(message);

    await db.collection("conversations").updateOne(
      { _id: new ObjectId(conversationId) },
      { $set: { lastMessage: { text, sentAt: message.sentAt } } }
    );

    io.to(conversationId).emit("new_message", { ...message, _id: result.insertedId });
  });

  socket.on("disconnect", () => {
    console.log(`User disconnected: ${socket.id}`);
  });
});
```

## Loading Message History

```javascript
app.get("/api/conversations/:id/messages", async (req, res) => {
  const { id } = req.params;
  const { before, limit = 50 } = req.query;

  const filter = {
    conversationId: new ObjectId(id),
    ...(before && { sentAt: { $lt: new Date(before) } })
  };

  const messages = await db.collection("messages")
    .find(filter)
    .sort({ sentAt: -1 })
    .limit(Number(limit))
    .toArray();

  res.json(messages.reverse());
});
```

## Tracking Read Status

```javascript
socket.on("mark_read", async ({ conversationId, userId }) => {
  await db.collection("messages").updateMany(
    { conversationId: new ObjectId(conversationId), readBy: { $ne: userId } },
    { $addToSet: { readBy: userId } }
  );

  socket.to(conversationId).emit("messages_read", { conversationId, userId });
});
```

## Summary

A MongoDB-backed chat application uses the `messages` collection for persistent storage and Socket.IO for real-time delivery. Store messages with conversation references, and use compound indexes on `conversationId` and `sentAt` for efficient message history queries. Use cursor-based pagination with `$lt: sentAt` to load older messages as users scroll up.
