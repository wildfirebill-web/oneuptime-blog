# How to Use MongoDB Atlas Serverless Instances

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Serverless, Cloud, Scaling

Description: Learn how to create and use MongoDB Atlas Serverless instances, understand the consumption-based pricing model, and connect from serverless functions and applications.

---

## Overview

MongoDB Atlas Serverless instances automatically scale compute and storage capacity up and down based on workload, charging only for the reads, writes, and storage you consume. This makes them ideal for development, intermittent workloads, and serverless applications.

## Creating an Atlas Serverless Instance

Via the Atlas UI:
1. Log in to MongoDB Atlas
2. Click "Build a Database"
3. Select "Serverless"
4. Choose a cloud provider (AWS, Azure, or GCP) and region
5. Name your instance and click "Create Instance"

Via the Atlas CLI:

```bash
atlas serverless create myServerlessInstance \
  --provider AWS \
  --region US_EAST_1
```

## Connecting to a Serverless Instance

The connection string looks identical to a standard cluster:

```bash
mongodb+srv://user:password@myserverlessinstance.abc123.mongodb.net/mydb
```

Connect with `mongosh`:

```bash
mongosh "mongodb+srv://user:password@myserverlessinstance.abc123.mongodb.net/mydb"
```

## Connecting from Node.js

```javascript
const { MongoClient } = require("mongodb");

const uri = process.env.MONGODB_URI;  // Atlas Serverless connection string

let cachedClient = null;

async function getDatabase() {
  if (!cachedClient) {
    cachedClient = new MongoClient(uri, {
      maxPoolSize: 1,
      serverSelectionTimeoutMS: 10000
    });
    await cachedClient.connect();
  }
  return cachedClient.db("mydb");
}

exports.handler = async () => {
  const db = await getDatabase();
  const count = await db.collection("orders").countDocuments();
  return { statusCode: 200, body: JSON.stringify({ count }) };
};
```

## Understanding Pricing

Atlas Serverless charges per operation:

- **Reads**: per million read processing units (RPUs)
- **Writes**: per million write processing units (WPUs)
- **Storage**: per GB-month

There is no charge for idle instances. A collection scan costs more RPUs than an index-covered query, so indexing is even more important with serverless pricing.

## Limitations vs Dedicated Clusters

- No sharding support (scale is vertical only, within limits)
- No custom indexes on `$text` search with Atlas Search (use Atlas Search instead)
- Maximum 500 connections per instance
- Not suitable for sustained high-throughput workloads

## Creating Indexes on a Serverless Instance

```javascript
// Create indexes just like on a standard cluster
db.orders.createIndex({ customerId: 1, createdAt: -1 })
db.orders.createIndex({ status: 1 })
```

## Monitoring Consumption

```bash
# View current usage via CLI
atlas serverless describe myServerlessInstance

# View metrics in Atlas UI: Metrics tab shows RPUs, WPUs, and storage consumed
```

## Summary

MongoDB Atlas Serverless instances provide on-demand capacity with no infrastructure management, making them ideal for applications with variable or unpredictable traffic. Connect with a standard driver and connection string. Optimize indexing carefully since each query's cost is directly tied to how many documents are scanned. For sustained high-throughput workloads, a dedicated cluster remains the better choice.
