# How to Use Debezium for MongoDB Change Data Capture

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Debezium, CDC, Kafka, Event-Driven

Description: Learn how to set up Debezium for MongoDB change data capture, publish change events to Kafka, and consume them in downstream services.

---

## Overview

Debezium is an open-source change data capture (CDC) platform that monitors MongoDB's oplog and publishes every insert, update, and delete as structured events to Apache Kafka. This enables real-time synchronization with other databases, caches, and search indexes without polling.

## Prerequisites

You need:
- A MongoDB replica set (Debezium requires oplog access)
- Apache Kafka and Kafka Connect
- Debezium MongoDB connector JAR

## Deploying the Debezium MongoDB Connector

Create a connector configuration file:

```json
{
  "name": "mongodb-debezium-connector",
  "config": {
    "connector.class": "io.debezium.connector.mongodb.MongoDbConnector",
    "mongodb.connection.string": "mongodb://mongo-user:mongo-pass@localhost:27017/?replicaSet=rs0",
    "mongodb.name": "myapp",
    "database.include.list": "mydb",
    "collection.include.list": "mydb.orders,mydb.customers",
    "snapshot.mode": "initial",
    "topic.prefix": "dbz"
  }
}
```

Register it via the Kafka Connect REST API:

```bash
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d @mongodb-debezium-connector.json
```

## Understanding the Event Format

Debezium wraps each change in an envelope with before/after document states:

```json
{
  "op": "c",
  "ts_ms": 1705000000000,
  "ns": { "db": "mydb", "coll": "orders" },
  "documentKey": { "_id": { "$oid": "64a1b2c3d4e5f6g7h8i9j0k1" } },
  "after": "{\"_id\": {\"$oid\": \"64a1b2c3d4e5f6g7h8i9j0k1\"}, \"status\": \"completed\", \"total\": 149.99}"
}
```

Operation types: `c` (insert), `u` (update), `d` (delete), `r` (read/snapshot).

## Consuming CDC Events in Node.js

```javascript
const { Kafka } = require("kafkajs");

const kafka = new Kafka({ brokers: ["localhost:9092"] });
const consumer = kafka.consumer({ groupId: "order-sync-service" });

await consumer.connect();
await consumer.subscribe({ topic: "dbz.mydb.orders", fromBeginning: false });

await consumer.run({
  eachMessage: async ({ message }) => {
    const event = JSON.parse(message.value.toString());
    const op = event.op;
    const doc = JSON.parse(event.after || "{}");

    if (op === "c") {
      console.log("New order inserted:", doc._id);
      await syncToSearch(doc);
    } else if (op === "u") {
      console.log("Order updated:", doc._id);
      await updateSearchIndex(doc);
    } else if (op === "d") {
      const key = JSON.parse(message.key.toString());
      console.log("Order deleted:", key._id);
      await removeFromSearch(key._id);
    }
  }
});
```

## Configuring the MongoDB User for Debezium

```javascript
db.createUser({
  user: "debezium",
  pwd: "secure-password",
  roles: [
    { role: "read", db: "mydb" },
    { role: "read", db: "local" },
    { role: "clusterMonitor", db: "admin" }
  ]
})
```

## Checking Connector Status

```bash
# Check connector status
curl http://localhost:8083/connectors/mongodb-debezium-connector/status

# Restart if failed
curl -X POST http://localhost:8083/connectors/mongodb-debezium-connector/restart
```

## Summary

Debezium captures MongoDB changes from the oplog and publishes them to Kafka topics, enabling real-time event-driven synchronization with downstream systems. Configure the connector with appropriate MongoDB user permissions, and consume the event envelope in downstream services to handle inserts, updates, and deletes. Use `snapshot.mode: initial` to capture the full initial state before streaming ongoing changes.
