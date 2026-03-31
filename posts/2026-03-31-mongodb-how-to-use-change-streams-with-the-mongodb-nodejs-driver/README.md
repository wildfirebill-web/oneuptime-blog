# How to Use Change Streams with the MongoDB Node.js Driver

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Change Streams, Node.js, Real-Time, Event-Driven

Description: Learn how to use MongoDB change streams in Node.js to watch for real-time insert, update, and delete events on collections, databases, or entire deployments.

---

## Overview

Change streams allow applications to subscribe to real-time change events from MongoDB collections, databases, or entire deployments. Built on the oplog, they provide a reliable, resumable stream of events without polling. Change streams require a replica set or sharded cluster (MongoDB Atlas works out of the box).

## Basic Collection Watch

```javascript
const { MongoClient } = require("mongodb");

const client = new MongoClient("mongodb://localhost:27017/?replicaSet=rs0");
await client.connect();

const collection = client.db("myapp").collection("orders");

// Watch all changes on the orders collection
const changeStream = collection.watch();

changeStream.on("change", (change) => {
  console.log("Change event:", JSON.stringify(change, null, 2));
});
```

## Change Event Types

| operationType | Triggered By |
|---|---|
| `insert` | insertOne, insertMany |
| `update` | updateOne, updateMany, findOneAndUpdate |
| `replace` | replaceOne |
| `delete` | deleteOne, deleteMany |
| `drop` | Collection dropped |
| `rename` | Collection renamed |
| `dropDatabase` | Database dropped |
| `invalidate` | Stream invalidated |

## Filtering Change Events with a Pipeline

Use an aggregation pipeline to filter the events you care about:

```javascript
const pipeline = [
  {
    $match: {
      operationType: { $in: ["insert", "update"] },
      "fullDocument.status": "pending"
    }
  }
];

const changeStream = collection.watch(pipeline, {
  fullDocument: "updateLookup"  // include the full document after update
});

changeStream.on("change", (change) => {
  console.log("New pending order:", change.fullDocument);
});
```

## Full Document on Update

By default, update events only include the delta (changed fields). Use `fullDocument: "updateLookup"` to also get the complete document:

```javascript
const changeStream = collection.watch([], {
  fullDocument: "updateLookup"
});

changeStream.on("change", (change) => {
  if (change.operationType === "update") {
    console.log("Updated fields:", change.updateDescription.updatedFields);
    console.log("Full document:", change.fullDocument);
  }
});
```

## Using async/await with for-await-of

```javascript
async function watchOrders(collection) {
  const pipeline = [{ $match: { operationType: "insert" } }];
  const changeStream = collection.watch(pipeline);

  try {
    for await (const change of changeStream) {
      console.log("New order inserted:", change.fullDocument);
      await processNewOrder(change.fullDocument);
    }
  } catch (err) {
    if (err.code === 40573) {
      console.log("Change stream closed");
    } else {
      throw err;
    }
  }
}
```

## Resuming a Change Stream After Failure

Change streams support resumption using a resume token. Store the token to resume from where you left off after a restart:

```javascript
const { MongoClient } = require("mongodb");
const fs = require("fs");

async function watchWithResume(collection) {
  let resumeToken = null;

  // Load saved resume token from disk
  if (fs.existsSync("./resume-token.json")) {
    resumeToken = JSON.parse(fs.readFileSync("./resume-token.json"));
  }

  const options = resumeToken ? { resumeAfter: resumeToken } : {};
  const changeStream = collection.watch([], options);

  changeStream.on("change", (change) => {
    // Save the resume token after each event
    fs.writeFileSync("./resume-token.json", JSON.stringify(change._id));
    console.log("Event:", change.operationType, change.documentKey);
  });

  changeStream.on("error", (err) => {
    console.error("Change stream error:", err);
  });
}
```

## Watching at the Database or Client Level

```javascript
// Watch all collections in a database
const dbStream = client.db("myapp").watch();

// Watch all collections in all databases
const clientStream = client.watch();

clientStream.on("change", (change) => {
  console.log(`Change in ${change.ns.db}.${change.ns.coll}:`, change.operationType);
});
```

## Practical Example: Real-Time Order Notifications

```javascript
const { MongoClient } = require("mongodb");
const notificationService = require("./notifications");

async function startOrderWatcher() {
  const client = new MongoClient(process.env.MONGODB_URI);
  await client.connect();

  const orders = client.db("store").collection("orders");

  const pipeline = [
    {
      $match: {
        operationType: "update",
        "updateDescription.updatedFields.status": { $exists: true }
      }
    }
  ];

  const changeStream = orders.watch(pipeline, { fullDocument: "updateLookup" });

  for await (const change of changeStream) {
    const { fullDocument: order } = change;
    await notificationService.notify(order.userId, {
      type: "ORDER_STATUS_CHANGED",
      orderId: order._id,
      newStatus: order.status
    });
  }
}
```

## Summary

MongoDB change streams provide a reliable, resumable mechanism for reacting to database events in real time. Use pipeline filters to narrow down the events your application receives, enable `fullDocument: "updateLookup"` when you need the complete document on updates, and always persist the resume token to survive restarts without missing events. Change streams eliminate the need for polling and are the foundation for event-driven architectures with MongoDB.
