# How to Handle MongoDB Failover Gracefully in Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Failover, Resilience

Description: Configure MongoDB driver settings and application patterns to handle replica set failovers gracefully without crashing or dropping requests.

---

## Understanding MongoDB Failover

When a MongoDB primary goes down, the replica set holds an election to promote a secondary. This process typically takes 10-30 seconds. During this window, write operations fail and reads may be disrupted depending on your read preference. Applications that are not configured for resilience will throw errors and potentially crash.

## Driver Configuration for Resilience

The most important settings are in your connection string and MongoClient options.

### Node.js (Mongoose)

```javascript
const mongoose = require('mongoose')

mongoose.connect('mongodb://mongo1:27017,mongo2:27017,mongo3:27017/myapp?replicaSet=rs0', {
  // Time to wait for a server to be selected before throwing
  serverSelectionTimeoutMS: 5000,
  // Time to wait for an operation to complete
  socketTimeoutMS: 45000,
  // Heartbeat frequency to detect failures quickly
  heartbeatFrequencyMS: 2000,
  // Allow reads from secondaries during failover
  readPreference: 'primaryPreferred',
  // Retry writes automatically once after a network error
  retryWrites: true,
  retryReads: true,
  // Connection pool size
  maxPoolSize: 50,
  minPoolSize: 5,
})
```

### Python (PyMongo)

```python
from pymongo import MongoClient, ReadPreference
from pymongo.errors import AutoReconnect, ConnectionFailure

client = MongoClient(
    'mongodb://mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0',
    serverSelectionTimeoutMS=5000,
    heartbeatFrequencyMS=2000,
    retryWrites=True,
    readPreference='primaryPreferred',
)
```

## Retry Logic

Implement a retry wrapper around critical write operations:

```javascript
async function withRetry(fn, maxRetries = 3, delayMs = 500) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn()
    } catch (err) {
      const isRetryable = err.name === 'MongoNetworkError' ||
                          err.name === 'MongoServerSelectionError' ||
                          err.message.includes('not primary')

      if (!isRetryable || attempt === maxRetries) throw err

      const backoff = delayMs * Math.pow(2, attempt - 1)
      console.warn(`Attempt ${attempt} failed, retrying in ${backoff}ms: ${err.message}`)
      await new Promise(r => setTimeout(r, backoff))
    }
  }
}

// Usage
await withRetry(() => db.orders.insertOne({ userId: '123', total: 49.99 }))
```

## Read Fallback During Failover

For non-critical reads, fall back to a secondary when the primary is unavailable:

```javascript
const { MongoClient, ReadPreference } = require('mongodb')

async function readWithFallback(collection, query) {
  try {
    return await collection.findOne(query, {
      readPreference: ReadPreference.PRIMARY,
    })
  } catch (err) {
    if (err.name === 'MongoServerSelectionError') {
      console.warn('Primary unavailable, reading from secondary')
      return collection.findOne(query, {
        readPreference: ReadPreference.SECONDARY_PREFERRED,
      })
    }
    throw err
  }
}
```

## Health Check Endpoint

Expose a health check that accounts for MongoDB state:

```javascript
app.get('/health', async (req, res) => {
  try {
    await mongoose.connection.db.admin().command({ ping: 1 })
    res.json({ status: 'ok', mongo: 'connected' })
  } catch (err) {
    res.status(503).json({ status: 'degraded', mongo: err.message })
  }
})
```

## Summary

Graceful MongoDB failover handling requires three layers: correct driver configuration (short heartbeat intervals, retry writes enabled, primaryPreferred read preference), application-level retry logic with exponential backoff for transient errors, and health endpoints that reflect the current database state. With these in place, a 20-second replica set election becomes a brief blip rather than a user-facing outage.
