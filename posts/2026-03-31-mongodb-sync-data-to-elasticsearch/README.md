# How to Sync MongoDB Data to Elasticsearch

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Elasticsearch, Sync, Search, CDC

Description: Learn how to sync MongoDB data to Elasticsearch in real time using Monstache, the MongoDB Kafka connector, or a custom change stream consumer for full-text search.

---

## Overview

Elasticsearch excels at full-text search and complex filtering, but MongoDB is better for transactional operations. Syncing MongoDB data to Elasticsearch lets you use each database for what it does best. There are three main approaches: Monstache (a dedicated sync tool), Kafka with Debezium, and a custom change stream consumer.

## Option 1: Using Monstache

Monstache is a Go-based tool that watches MongoDB change streams and syncs documents to Elasticsearch in real time.

Install and configure `monstache`:

```toml
# monstache.toml
mongo-url = "mongodb://localhost:27017"
elasticsearch-urls = ["http://localhost:9200"]

[[mapping]]
namespace = "mydb.products"
index = "products"

[[mapping]]
namespace = "mydb.orders"
index = "orders"

[gzip]
enabled = true
```

Run it:

```bash
monstache -f monstache.toml
```

## Option 2: Kafka with Debezium and Elasticsearch Sink

Set up a Debezium MongoDB source connector and an Elasticsearch sink connector:

```json
// Elasticsearch sink connector config
{
  "name": "elasticsearch-sink",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "tasks.max": "1",
    "topics": "dbz.mydb.products",
    "connection.url": "http://elasticsearch:9200",
    "type.name": "_doc",
    "key.ignore": "false",
    "schema.ignore": "true"
  }
}
```

## Option 3: Custom Change Stream Consumer in Node.js

```javascript
const { MongoClient } = require("mongodb");
const { Client } = require("@elastic/elasticsearch");

const mongo = new MongoClient("mongodb://localhost:27017");
const es = new Client({ node: "http://localhost:9200" });

async function startSync() {
  await mongo.connect();
  const collection = mongo.db("mydb").collection("products");
  const changeStream = collection.watch([], { fullDocument: "updateLookup" });

  for await (const change of changeStream) {
    const id = change.documentKey._id.toString();

    if (change.operationType === "insert" || change.operationType === "update") {
      const doc = change.fullDocument;
      await es.index({
        index: "products",
        id: id,
        document: {
          name: doc.name,
          description: doc.description,
          category: doc.category,
          price: doc.price,
          updatedAt: new Date()
        }
      });
      console.log(`Synced ${change.operationType} for product ${id}`);
    } else if (change.operationType === "delete") {
      await es.delete({ index: "products", id: id });
      console.log(`Deleted product ${id} from Elasticsearch`);
    }
  }
}

startSync().catch(console.error);
```

## Initial Bulk Sync

Before streaming changes, do an initial bulk load from MongoDB:

```javascript
async function initialSync() {
  const cursor = mongo.db("mydb").collection("products").find({});
  const operations = [];

  for await (const doc of cursor) {
    operations.push({ index: { _index: "products", _id: doc._id.toString() } });
    operations.push({
      name: doc.name,
      description: doc.description,
      category: doc.category,
      price: doc.price
    });

    if (operations.length >= 1000) {
      await es.bulk({ operations });
      operations.length = 0;
    }
  }

  if (operations.length > 0) await es.bulk({ operations });
  console.log("Initial sync complete");
}
```

## Summary

Sync MongoDB data to Elasticsearch using Monstache for a zero-code approach, Kafka with Debezium for a scalable event pipeline, or a custom Node.js change stream consumer for full control over field mapping. Always do an initial bulk sync before starting the change stream to ensure Elasticsearch has a complete copy of existing data.
