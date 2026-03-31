# How to Use $changeStream in MongoDB Aggregation Pipelines

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Change Stream, Aggregation, Real-Time

Description: Learn how to open and filter MongoDB change streams using aggregation pipeline stages, including watching collections, databases, and entire deployments for real-time events.

---

## What Are Change Streams?

Change streams allow applications to subscribe to real-time data changes in MongoDB collections, databases, or entire deployments. They are built on top of the oplog and are available on replica sets and sharded clusters. Aggregation pipeline stages can be applied to filter and transform the change events before they reach your application.

## Opening a Change Stream

```javascript
// Watch a collection
const changeStream = db.collection("orders").watch()

// Watch a specific database
const dbStream = db.watch()

// Watch the entire deployment (admin database)
const deploymentStream = client.watch()
```

## Filtering Change Events with a Pipeline

Pass an aggregation pipeline as the first argument to `watch()`.

```javascript
const pipeline = [
  { $match: { operationType: "insert" } }
]

const changeStream = db.collection("orders").watch(pipeline)

for await (const change of changeStream) {
  console.log("New order:", change.fullDocument)
}
```

## Filtering by Operation Type

```javascript
const pipeline = [
  {
    $match: {
      operationType: { $in: ["insert", "update"] }
    }
  }
]
```

Available operation types: `insert`, `update`, `replace`, `delete`, `drop`, `rename`, `dropDatabase`, `invalidate`.

## Filtering by Document Field Values

```javascript
const pipeline = [
  {
    $match: {
      operationType: "insert",
      "fullDocument.status": "urgent",
      "fullDocument.amount": { $gte: 1000 }
    }
  }
]

const stream = db.collection("orders").watch(pipeline, {
  fullDocument: "updateLookup"
})
```

Note: `fullDocument` is only available by default for `insert` and `replace`. For `update`, set the `fullDocument` option to `"updateLookup"`.

## Watching for Changes to Specific Fields

```javascript
const pipeline = [
  { $match: { operationType: "update" } },
  {
    $match: {
      "updateDescription.updatedFields.status": { $exists: true }
    }
  }
]
```

## Projecting Change Event Fields

```javascript
const pipeline = [
  {
    $project: {
      operationType: 1,
      "fullDocument._id": 1,
      "fullDocument.userId": 1,
      "fullDocument.action": 1,
      clusterTime: 1
    }
  }
]
```

## Resuming a Change Stream

Change streams return a resume token with each event. Store it and use it to resume after a disconnect.

```javascript
let resumeToken = null

const stream = db.collection("events").watch(pipeline)

for await (const change of stream) {
  resumeToken = change._id
  await processChange(change)
}

// On reconnect:
const resumedStream = db.collection("events").watch(pipeline, {
  resumeAfter: resumeToken
})
```

## Listening for All Changes in a Database

```javascript
const dbPipeline = [
  {
    $match: {
      "ns.coll": { $in: ["orders", "payments"] }
    }
  }
]

const stream = db.watch(dbPipeline)
```

## Full Document Pre and Post Images

MongoDB 6.0+ supports capturing the document state before and after an update.

```javascript
// Enable pre and post images on the collection
db.runCommand({
  collMod: "orders",
  changeStreamPreAndPostImages: { enabled: true }
})

// Watch with both images
const stream = db.collection("orders").watch(pipeline, {
  fullDocument: "whenAvailable",
  fullDocumentBeforeChange: "whenAvailable"
})
```

## Summary

Change streams with aggregation pipelines let you subscribe to specific subsets of MongoDB change events in real time. Use `$match` to filter by operation type or document fields, `$project` to trim event payloads, and always store the resume token so your application can reconnect without missing events after a disruption.
