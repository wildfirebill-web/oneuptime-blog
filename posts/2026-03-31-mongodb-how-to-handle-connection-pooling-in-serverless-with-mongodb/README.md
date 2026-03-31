# How to Handle Connection Pooling in Serverless with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mongodb, Serverless, Connection Pooling, Lambda, Performance

Description: Solve MongoDB connection exhaustion in serverless environments like AWS Lambda and Vercel by reusing connections and using MongoDB Atlas connection pooling.

---

## Overview

Serverless functions like AWS Lambda, Vercel Functions, and Google Cloud Functions spin up thousands of instances simultaneously, each trying to open its own MongoDB connection. Without proper connection management, you can exhaust MongoDB's connection limit (by default 100,000 connections on Atlas but much lower on self-hosted). This guide covers strategies to handle MongoDB connections efficiently in serverless environments.

## The Problem

Traditional connection pooling assumes a long-running process. In serverless:

- Each function instance opens a new connection by default
- Connections are not shared between instances
- Cold starts re-open connections
- During traffic spikes, thousands of connections open simultaneously

```javascript
// BAD - opens a new connection on every invocation
exports.handler = async (event) => {
  const client = new MongoClient(process.env.MONGODB_URI)
  await client.connect()
  const db = client.db("myapp")
  const result = await db.collection("users").findOne({ id: event.userId })
  await client.close()  // closes connection, wastes the next warm start
  return result
}
```

## Solution 1 - Reuse Connection Across Warm Invocations

```javascript
// GOOD - reuse connection across warm Lambda invocations
const { MongoClient } = require("mongodb")

let cachedClient = null
let cachedDb = null

async function connectToDatabase() {
  if (cachedClient && cachedDb) {
    return { client: cachedClient, db: cachedDb }
  }

  const client = new MongoClient(process.env.MONGODB_URI, {
    maxPoolSize: 10,          // max 10 connections per Lambda instance
    serverSelectionTimeoutMS: 5000,
    socketTimeoutMS: 45000,
    connectTimeoutMS: 10000,
    maxIdleTimeMS: 270000     // close idle connections after 4.5 minutes
  })

  await client.connect()
  cachedClient = client
  cachedDb = client.db("myapp")

  return { client: cachedClient, db: cachedDb }
}

exports.handler = async (event) => {
  const { db } = await connectToDatabase()
  const result = await db.collection("users").findOne({ id: event.userId })
  return result
  // Do NOT close the connection - reuse it on the next warm invocation
}
```

## Solution 2 - Use MongoDB Atlas Data API

For very high concurrency with minimal connection overhead, use the Atlas Data API (HTTP-based, no persistent connections):

```javascript
// No persistent connections - each request is stateless HTTP
const ATLAS_DATA_API_URL = process.env.ATLAS_DATA_API_URL
const ATLAS_API_KEY = process.env.ATLAS_API_KEY

async function findUser(userId) {
  const response = await fetch(`${ATLAS_DATA_API_URL}/action/findOne`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "api-key": ATLAS_API_KEY
    },
    body: JSON.stringify({
      dataSource: "Cluster0",
      database: "myapp",
      collection: "users",
      filter: { id: userId }
    })
  })
  const data = await response.json()
  return data.document
}

exports.handler = async (event) => {
  const user = await findUser(event.userId)
  return user
}
```

## Solution 3 - MongoDB Atlas Connection Pooling Proxy

Atlas provides a built-in connection pooler. Enable it in the Atlas UI or via the CLI:

```bash
# Enable connection pooling in Atlas cluster settings
# Connection string format with pooling proxy:
# mongodb+srv://user:pass@cluster.mongodb.net/?maxPoolSize=10
```

Configure your serverless function to use a reduced pool:

```javascript
const client = new MongoClient(process.env.MONGODB_URI, {
  maxPoolSize: 5,     // small pool per instance
  minPoolSize: 1,
  waitQueueTimeoutMS: 5000
})
```

## Solution 4 - Mongoose with Serverless

If using Mongoose, prevent re-connecting on warm invocations:

```javascript
const mongoose = require("mongoose")

let isConnected = false

async function dbConnect() {
  if (isConnected) {
    console.log("Using existing MongoDB connection")
    return
  }

  await mongoose.connect(process.env.MONGODB_URI, {
    maxPoolSize: 10,
    serverSelectionTimeoutMS: 5000,
    socketTimeoutMS: 45000,
    bufferCommands: false,
    bufferTimeoutMS: 20000
  })

  isConnected = mongoose.connection.readyState === 1
  console.log("New MongoDB connection established")
}

// Use in serverless function
module.exports.handler = async (event) => {
  await dbConnect()
  const User = mongoose.model("User", userSchema)
  return User.findById(event.userId)
}
```

## Solution 5 - Vercel/Next.js API Routes

```javascript
// lib/mongodb.js
import { MongoClient } from "mongodb"

const uri = process.env.MONGODB_URI
const options = {
  maxPoolSize: 10,
  serverSelectionTimeoutMS: 5000
}

let client
let clientPromise

if (process.env.NODE_ENV === "development") {
  // In development, use a global variable to preserve the connection
  if (!global._mongoClientPromise) {
    client = new MongoClient(uri, options)
    global._mongoClientPromise = client.connect()
  }
  clientPromise = global._mongoClientPromise
} else {
  // In production, create a module-level cached promise
  client = new MongoClient(uri, options)
  clientPromise = client.connect()
}

export default clientPromise

// pages/api/users/[id].js
import clientPromise from "../../../lib/mongodb"

export default async function handler(req, res) {
  const client = await clientPromise
  const db = client.db("myapp")

  const user = await db.collection("users").findOne({
    _id: req.query.id
  })

  res.json(user)
}
```

## Monitor Connection Usage

```javascript
// Log connection pool stats
const client = new MongoClient(uri, {
  monitorCommands: true
})

client.on("connectionPoolCreated", (event) => {
  console.log("Pool created:", event)
})

client.on("connectionCreated", (event) => {
  console.log("New connection:", event.connectionId)
})

client.on("connectionClosed", (event) => {
  console.log("Connection closed:", event.connectionId)
})
```

Check connection count in MongoDB Atlas or via shell:

```javascript
db.serverStatus().connections
// { current: 45, available: 49955, totalCreated: 1200 }
```

## Summary

Handling MongoDB connections in serverless requires caching the MongoClient instance at the module level so warm function invocations reuse existing connections rather than opening new ones. Set `maxPoolSize` low (5-10) per instance to avoid overwhelming MongoDB during spikes, configure `maxIdleTimeMS` to clean up connections after periods of inactivity, and consider the Atlas Data API for extreme concurrency scenarios where HTTP-based access is preferable to persistent TCP connections.
