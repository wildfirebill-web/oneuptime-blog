# How to Use MongoDB Change Streams with Apache Kafka

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Kafka, Change Stream, CDC, Event-Driven

Description: Learn how to connect MongoDB change streams to Apache Kafka using the official Kafka connector and the Node.js driver to build event-driven data pipelines.

---

## Overview

MongoDB change streams let you subscribe to real-time data changes. By connecting change streams to Apache Kafka, you can build event-driven architectures where downstream services react to database changes without polling.

## Option 1: MongoDB Kafka Connector

The official MongoDB Kafka Connector publishes change events directly to Kafka topics without writing application code.

Deploy the connector config:

```json
{
  "name": "mongodb-source-connector",
  "config": {
    "connector.class": "com.mongodb.kafka.connect.MongoSourceConnector",
    "connection.uri": "mongodb://localhost:27017",
    "database": "mydb",
    "collection": "orders",
    "topic.prefix": "mongo",
    "publish.full.document.only": "true",
    "change.stream.full.document": "updateLookup"
  }
}
```

Deploy via Kafka Connect REST API:

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @mongodb-source-connector.json
```

This publishes change events to the `mongo.mydb.orders` Kafka topic.

## Option 2: Custom Change Stream Producer in Node.js

For more control over event transformation, write a custom producer:

```javascript
const { MongoClient } = require("mongodb");
const { Kafka } = require("kafkajs");

const mongo = new MongoClient("mongodb://localhost:27017");
const kafka = new Kafka({ brokers: ["localhost:9092"] });
const producer = kafka.producer();

async function startChangeStreamProducer() {
  await mongo.connect();
  await producer.connect();

  const collection = mongo.db("mydb").collection("orders");
  const changeStream = collection.watch([], {
    fullDocument: "updateLookup"
  });

  console.log("Watching for changes...");
  for await (const change of changeStream) {
    const event = {
      operationType: change.operationType,
      documentKey: change.documentKey,
      fullDocument: change.fullDocument || null,
      timestamp: change.clusterTime
    };

    await producer.send({
      topic: "order-events",
      messages: [{ key: change.documentKey._id.toString(), value: JSON.stringify(event) }]
    });

    console.log(`Published ${change.operationType} event for ${change.documentKey._id}`);
  }
}

startChangeStreamProducer().catch(console.error);
```

## Handling Resume Tokens

Change streams provide a resume token so your consumer can resume from where it left off after a restart:

```javascript
let resumeToken = null;

// Load persisted token on startup
const tokenDoc = await db.collection("stream_state").findOne({ _id: "orders_stream" });
if (tokenDoc) resumeToken = tokenDoc.token;

const changeStream = collection.watch([], {
  resumeAfter: resumeToken,
  fullDocument: "updateLookup"
});

for await (const change of changeStream) {
  await processChange(change);

  // Persist the resume token after each successful event
  resumeToken = change._id;
  await db.collection("stream_state").updateOne(
    { _id: "orders_stream" },
    { $set: { token: resumeToken } },
    { upsert: true }
  );
}
```

## Consuming the Kafka Topic

```javascript
const consumer = kafka.consumer({ groupId: "order-processor" });

await consumer.connect();
await consumer.subscribe({ topic: "order-events" });

await consumer.run({
  eachMessage: async ({ message }) => {
    const event = JSON.parse(message.value.toString());
    console.log(`Processing ${event.operationType} for order ${event.documentKey._id}`);
    // Handle the event downstream
  }
});
```

## Summary

MongoDB change streams can be connected to Apache Kafka using the official MongoDB Kafka Connector for zero-code deployment or a custom Node.js producer for fine-grained control. Always persist the resume token so your producer can recover from restarts without missing events. This pattern enables event-driven microservices that react to database changes in real time.
