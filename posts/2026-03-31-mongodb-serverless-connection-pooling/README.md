# How to Handle Connection Pooling in Serverless with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Serverless, Connection Pool, Lambda, Performance

Description: Learn how to manage MongoDB connection pooling in serverless environments like AWS Lambda and Vercel to avoid connection exhaustion and cold start latency.

---

## Overview

Serverless functions create a new execution environment per invocation, which can exhaust MongoDB's connection limit if each function opens a new connection. The solution is to cache the connection outside the handler function and use MongoDB Atlas or a connection pooler like connection proxying.

## The Problem: Connection Exhaustion

A naive Lambda function opens a new connection on every invocation:

```javascript
// BAD: Creates a new connection on every invocation
exports.handler = async (event) => {
  const client = new MongoClient(process.env.MONGODB_URI);
  await client.connect();  // New connection each time
  const result = await client.db("mydb").collection("orders").findOne({});
  await client.close();
  return result;
};
```

With thousands of concurrent invocations, this exhausts MongoDB's default connection limit of 200.

## Solution: Cache the Connection Outside the Handler

Node.js Lambda execution environments are reused across warm invocations. Declare the client and connection outside the handler:

```javascript
const { MongoClient } = require("mongodb");

let cachedClient = null;
let cachedDb = null;

async function connectToDatabase() {
  if (cachedClient && cachedClient.topology?.isConnected()) {
    return cachedDb;
  }

  const client = new MongoClient(process.env.MONGODB_URI, {
    maxPoolSize: 10,
    serverSelectionTimeoutMS: 5000
  });

  await client.connect();
  cachedClient = client;
  cachedDb = client.db("mydb");
  return cachedDb;
}

exports.handler = async (event) => {
  const db = await connectToDatabase();
  const result = await db.collection("orders").findOne({ status: "pending" });
  return { statusCode: 200, body: JSON.stringify(result) };
};
```

## Configuring Pool Size for Serverless

Set `maxPoolSize` low (1-5) in serverless environments since many functions run in parallel rather than sequentially:

```javascript
const client = new MongoClient(uri, {
  maxPoolSize: 5,
  minPoolSize: 1,
  maxIdleTimeMS: 10000,
  waitQueueTimeoutMS: 5000,
  serverSelectionTimeoutMS: 5000
});
```

## Using MongoDB Atlas Serverless or Data API

For extreme scale, MongoDB Atlas Serverless instances scale connections automatically. Alternatively, the Atlas Data API handles connections on behalf of your functions via HTTP - no driver or connection management needed:

```javascript
// Atlas Data API - no connection pooling needed
const response = await fetch(
  `https://data.mongodb-api.com/app/${process.env.APP_ID}/endpoint/data/v1/action/findOne`,
  {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "api-key": process.env.DATA_API_KEY
    },
    body: JSON.stringify({
      dataSource: "Cluster0",
      database: "mydb",
      collection: "orders",
      filter: { status: "pending" }
    })
  }
);
```

## Next.js API Routes

The same pattern applies to Next.js serverless API routes:

```javascript
// lib/mongodb.js
import { MongoClient } from "mongodb";

let client;
let clientPromise;

if (process.env.NODE_ENV === "development") {
  if (!global._mongoClientPromise) {
    client = new MongoClient(process.env.MONGODB_URI);
    global._mongoClientPromise = client.connect();
  }
  clientPromise = global._mongoClientPromise;
} else {
  client = new MongoClient(process.env.MONGODB_URI);
  clientPromise = client.connect();
}

export default clientPromise;
```

## Summary

In serverless environments, cache the MongoDB client outside the handler function to reuse connections across warm invocations. Set `maxPoolSize` to 1-5 to avoid overwhelming MongoDB with parallel function executions. For massive scale, use MongoDB Atlas Serverless instances or the Atlas Data API to offload connection management entirely.
