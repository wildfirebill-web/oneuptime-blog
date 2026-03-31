# How to Build a Data Warehouse Layer on MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Data Warehouse, Analytics, Aggregation, Schema

Description: Learn how to build a data warehouse layer on MongoDB using materialized aggregations, denormalized reporting collections, and scheduled pipeline jobs for fast analytical queries.

---

MongoDB is a capable platform for building a lightweight analytical layer alongside your operational data. By creating pre-aggregated reporting collections using the aggregation pipeline, you can serve dashboard queries in milliseconds without moving data to a separate warehouse.

## The Pattern: Materialized Aggregations

Instead of running expensive aggregations on demand, pre-compute them on a schedule and store the results in dedicated reporting collections. Applications query the reporting collections directly.

```text
Raw Collections     -->  Aggregation Pipeline  -->  Reporting Collections
orders                   (scheduled daily)          daily_revenue_summary
events                                              monthly_user_cohorts
sessions                                            product_performance
```

## Create a Daily Revenue Summary

```javascript
db.orders.aggregate([
  {
    $match: {
      status: "completed",
      createdAt: {
        $gte: new Date(new Date().setHours(0, 0, 0, 0) - 86400000),
        $lt: new Date(new Date().setHours(0, 0, 0, 0))
      }
    }
  },
  {
    $group: {
      _id: {
        date: { $dateToString: { format: "%Y-%m-%d", date: "$createdAt" } },
        category: "$category",
        region: "$region"
      },
      revenue: { $sum: "$amount" },
      orders: { $sum: 1 },
      uniqueCustomers: { $addToSet: "$customerId" }
    }
  },
  {
    $addFields: {
      uniqueCustomerCount: { $size: "$uniqueCustomers" }
    }
  },
  { $unset: "uniqueCustomers" },
  { $merge: { into: "daily_revenue_summary", on: "_id", whenMatched: "replace", whenNotMatched: "insert" } }
])
```

The `$merge` stage writes results directly to the reporting collection, upserting existing records and inserting new ones.

## Add Indexes to Reporting Collections

Create indexes on the dimensions most frequently used in dashboard filters:

```javascript
db.daily_revenue_summary.createIndex({ "_id.date": -1 })
db.daily_revenue_summary.createIndex({ "_id.category": 1, "_id.date": -1 })
db.daily_revenue_summary.createIndex({ "_id.region": 1, "_id.date": -1 })
```

## Build a Cohort Analysis Table

Group users by their acquisition month and track retention:

```javascript
db.users.aggregate([
  {
    $group: {
      _id: {
        cohort: { $dateToString: { format: "%Y-%m", date: "$createdAt" } }
      },
      users: { $addToSet: "$_id" },
      count: { $sum: 1 }
    }
  },
  { $merge: { into: "user_cohorts", on: "_id", whenMatched: "replace", whenNotMatched: "insert" } }
])
```

## Schedule the Aggregation Jobs

Use a simple Node.js script scheduled with cron:

```javascript
const { MongoClient } = require("mongodb")

async function runDailyAggregations() {
  const client = new MongoClient(process.env.MONGO_URI)
  await client.connect()
  const db = client.db("salesdb")

  console.log("Running daily revenue aggregation...")
  await db.collection("orders").aggregate(dailyRevenuePipeline).toArray()

  console.log("Running cohort aggregation...")
  await db.collection("users").aggregate(cohortPipeline).toArray()

  await client.close()
}

runDailyAggregations().catch(console.error)
```

Cron schedule:

```text
0 1 * * * node /opt/aggregations/run_daily.js
```

## Query the Reporting Layer

Dashboard queries against the reporting collections are simple and fast:

```javascript
db.daily_revenue_summary.find({
  "_id.date": { $gte: "2025-01-01" },
  "_id.region": "us-east"
}).sort({ "_id.date": -1 })
```

## Summary

Building a data warehouse layer on MongoDB uses pre-computed aggregation collections refreshed on a schedule. The `$merge` stage handles upserts cleanly, indexes on reporting collections keep dashboard queries fast, and scheduled scripts keep the data fresh. This approach avoids the complexity of a separate warehouse for moderate analytical workloads.
