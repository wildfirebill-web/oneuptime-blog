# How to Use MongoDB Atlas Serverless Instances

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mongodb, Atlas Serverless, Cloud, Auto-Scaling, Serverless

Description: Learn how to create and use MongoDB Atlas Serverless instances that automatically scale to zero and charge only for operations you perform.

---

## Overview

MongoDB Atlas Serverless instances are a pay-per-use deployment option where you are billed based on the number of read and write processing units (RPUs/WPUs) consumed, not on reserved capacity. They automatically scale from zero to handle traffic spikes and back to zero during idle periods, making them ideal for development environments, low-traffic applications, and workloads with unpredictable traffic.

## Serverless vs Dedicated Clusters

| Feature | Serverless | Dedicated Cluster |
|---------|-----------|-------------------|
| Billing | Per operation (RPU/WPU) | Hourly (reserved capacity) |
| Scaling | Automatic, to zero | Manual or auto-scaling |
| Use case | Low/variable traffic | Consistent high traffic |
| Change Streams | Limited | Full support |
| Transactions | Supported | Supported |
| Connections | Shared proxy | Dedicated |

## Step 1 - Create a Serverless Instance

Using the Atlas CLI:

```bash
# Install Atlas CLI
brew install mongodb-atlas-cli

# Log in
atlas auth login

# Create serverless instance
atlas serverless create my-serverless \
  --provider AWS \
  --region US_EAST_1

# Check status
atlas serverless describe my-serverless
```

Using Atlas UI: Navigate to Database - Create - Serverless.

## Step 2 - Get the Connection String

```bash
# Get connection details
atlas serverless describe my-serverless

# Output includes:
# connectionStrings.standardSrv: mongodb+srv://...
```

The serverless connection string looks like:

```text
mongodb+srv://<username>:<password>@my-serverless.abc123.mongodb.net/?retryWrites=true&w=majority
```

## Step 3 - Connect from Node.js

```javascript
const { MongoClient } = require("mongodb")

const uri = process.env.MONGODB_URI  // Atlas serverless connection string
const client = new MongoClient(uri, {
  // Serverless instances use a shared proxy - keep pool small
  maxPoolSize: 10,
  serverSelectionTimeoutMS: 10000,
  retryWrites: true,
  retryReads: true
})

let db

async function connect() {
  if (!db) {
    await client.connect()
    db = client.db("myapp")
  }
  return db
}

// Usage
async function getUser(id) {
  const db = await connect()
  return db.collection("users").findOne({ _id: id })
}
```

## Step 4 - Understand Serverless Billing

Processing units are consumed per operation:

```text
Reads (RPUs):
- findOne: ~0.5 RPU
- find (returns 1000 docs): ~100 RPUs
- aggregate pipeline: variable

Writes (WPUs):
- insertOne: ~1 WPU
- updateOne: ~1 WPU
- deleteOne: ~0.5 WPU

Storage: $0.25 per GB/month (first 1GB free)
```

Estimate costs with Atlas Cost Explorer or use the billing API:

```bash
atlas billing invoices list
```

## Step 5 - Optimize Queries for Cost Efficiency

On serverless, each document scanned costs RPUs even if not returned. Use indexes aggressively:

```javascript
// EXPENSIVE without index - scans all documents
db.orders.find({ status: "pending", userId: "u123" })

// Create compound index to avoid collection scan
db.orders.createIndex({ userId: 1, status: 1 })

// CHEAP - index-covered query
db.orders.find(
  { userId: "u123", status: "pending" },
  { projection: { _id: 1, total: 1, createdAt: 1 } }  // covered by index
)
```

Projection reduces data scanned:

```javascript
// Returns only needed fields - reduces RPUs
const user = await db.collection("users").findOne(
  { email: "alice@example.com" },
  { projection: { name: 1, email: 1, _id: 0 } }
)
```

## Step 6 - Use Atlas Serverless with Vercel

```javascript
// lib/mongodb.js (Vercel/Next.js)
import { MongoClient } from "mongodb"

const uri = process.env.MONGODB_URI

let client
let clientPromise

if (process.env.NODE_ENV === "development") {
  if (!global._mongoClientPromise) {
    client = new MongoClient(uri)
    global._mongoClientPromise = client.connect()
  }
  clientPromise = global._mongoClientPromise
} else {
  client = new MongoClient(uri)
  clientPromise = client.connect()
}

export default clientPromise
```

```javascript
// pages/api/data.js
import clientPromise from "../../lib/mongodb"

export default async function handler(req, res) {
  const client = await clientPromise
  const db = client.db("myapp")

  const data = await db.collection("items").find({}).limit(20).toArray()
  res.json(data)
}
```

## Step 7 - Serverless Limitations to Know

Serverless instances have some limitations compared to dedicated clusters:

```javascript
// Change Streams - limited availability (check Atlas docs for current status)
// Cannot use $where operator
// Cannot use $currentOp

// These work fine on serverless:
const result = await db.collection("products").aggregate([
  { $match: { category: "electronics" } },
  { $group: { _id: "$brand", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
]).toArray()

// Multi-document transactions work
const session = client.startSession()
await session.withTransaction(async () => {
  await db.collection("accounts").updateOne(
    { _id: "acc-1" }, { $inc: { balance: -100 } }, { session }
  )
  await db.collection("accounts").updateOne(
    { _id: "acc-2" }, { $inc: { balance: 100 } }, { session }
  )
})
```

## Step 8 - Monitor Serverless Usage

```bash
# View current month's processing units
atlas serverless describe my-serverless

# Check Atlas UI: Database - my-serverless - Metrics tab
# Shows RPUs, WPUs, storage, and connections over time
```

Set billing alerts in Atlas UI: Billing - Alert Settings - Add Alert.

## Summary

MongoDB Atlas Serverless instances eliminate infrastructure management by automatically scaling to demand and charging only for operations performed. They are ideal for development, staging, and production workloads with variable or low traffic. To minimize costs, ensure queries use indexes to avoid collection scans, use projections to return only needed fields, and cache the MongoClient connection at the module level to avoid reconnection overhead on warm serverless function invocations.
