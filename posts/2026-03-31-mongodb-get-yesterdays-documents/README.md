# How to Get Yesterday's Documents in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query, Date, Filter, Time Range

Description: Learn how to query MongoDB for documents created or updated yesterday using date range filters with $gte and $lt operators in the shell and application code.

---

Fetching documents from the previous day is a common reporting requirement - daily summaries, overnight batch jobs, and scheduled reports all need to target a precise 24-hour window. MongoDB makes this straightforward with date range queries.

## Calculate Yesterday's Range

Yesterday spans from midnight to midnight (exclusive) in UTC:

```javascript
// In mongosh
const today = new Date()
today.setUTCHours(0, 0, 0, 0)

const yesterday = new Date(today)
yesterday.setUTCDate(today.getUTCDate() - 1)

print("Yesterday start:", yesterday)
print("Yesterday end:", today)
```

## Query for Yesterday's Documents

```javascript
const today = new Date()
today.setUTCHours(0, 0, 0, 0)

const yesterday = new Date(today)
yesterday.setUTCDate(today.getUTCDate() - 1)

db.orders.find({
  createdAt: {
    $gte: yesterday,
    $lt: today
  }
})
```

The `$gte` / `$lt` pattern gives you the exact 24-hour window without including documents from today.

## Count Yesterday's Documents

```javascript
const count = db.orders.countDocuments({
  createdAt: { $gte: yesterday, $lt: today }
})
print(`Orders yesterday: ${count}`)
```

## Aggregate Yesterday's Data

Summarize revenue from yesterday by category:

```javascript
db.orders.aggregate([
  {
    $match: {
      status: "completed",
      createdAt: { $gte: yesterday, $lt: today }
    }
  },
  {
    $group: {
      _id: "$category",
      revenue: { $sum: "$amount" },
      count: { $sum: 1 }
    }
  },
  { $sort: { revenue: -1 } }
])
```

## Python Example

```python
from datetime import datetime, timezone, timedelta
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017/")
db = client["salesdb"]

# Calculate yesterday's UTC range
now = datetime.now(timezone.utc)
today_start = now.replace(hour=0, minute=0, second=0, microsecond=0)
yesterday_start = today_start - timedelta(days=1)

orders = list(db.orders.find({
    "createdAt": {
        "$gte": yesterday_start,
        "$lt": today_start
    }
}))

print(f"Found {len(orders)} orders from yesterday")
```

## Node.js Example

```javascript
const { MongoClient } = require("mongodb")

async function getYesterdaysOrders() {
  const client = new MongoClient(process.env.MONGO_URI)
  await client.connect()

  const db = client.db("salesdb")

  const today = new Date()
  today.setUTCHours(0, 0, 0, 0)

  const yesterday = new Date(today)
  yesterday.setUTCDate(today.getUTCDate() - 1)

  const orders = await db.collection("orders").find({
    createdAt: { $gte: yesterday, $lt: today }
  }).toArray()

  console.log(`Found ${orders.length} orders from yesterday`)
  await client.close()
}
```

## Index for Performance

Ensure the date field is indexed to avoid collection scans:

```javascript
db.orders.createIndex({ createdAt: -1 })
```

For queries that always include a status filter, create a compound index:

```javascript
db.orders.createIndex({ status: 1, createdAt: -1 })
```

Verify the index is being used:

```javascript
db.orders.find({ createdAt: { $gte: yesterday, $lt: today } }).explain("executionStats")
```

## Handle Timezones

If your application stores timestamps in a local timezone rather than UTC, adjust the range calculation accordingly. The recommended practice is to store all timestamps as UTC in MongoDB and convert to local time only at the presentation layer.

## Summary

Querying for yesterday's documents in MongoDB uses a `$gte` / `$lt` date range spanning midnight to midnight UTC. Always calculate the range programmatically to avoid hardcoded dates. Create an index on the date field for performance, and prefer storing timestamps in UTC to simplify range queries across timezones.
