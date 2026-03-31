# How to Optimize $lookup Performance in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Lookup, Performance, Index

Description: Learn how to speed up MongoDB $lookup joins by indexing the foreign collection, reducing the pipeline input, and using the pipeline form of $lookup for complex joins.

---

`$lookup` is a powerful aggregation stage that performs a left outer join between collections. Without proper indexing it can be extremely slow, doing full collection scans on the "joined" collection for every document in the pipeline.

## The Root Performance Problem

By default, `$lookup` uses the `from` collection's `_id` field if the `foreignField` is `_id`, but for any other field it performs a scan unless an index exists.

```javascript
// Slow: no index on orders.customerId
db.customers.aggregate([
  {
    $lookup: {
      from: "orders",
      localField: "_id",
      foreignField: "customerId",
      as: "orders"
    }
  }
])
```

## Fix 1 - Index the Foreign Field

Create an index on the foreign collection's join field:

```javascript
db.orders.createIndex({ customerId: 1 })
```

Now the `$lookup` performs an `IXSCAN` on `orders` instead of a `COLLSCAN` for each customer document.

## Fix 2 - Filter Before $lookup

Reduce the number of documents entering `$lookup` with an early `$match`:

```javascript
db.customers.aggregate([
  { $match: { status: "active", region: "US" } },   // filter first
  {
    $lookup: {
      from: "orders",
      localField: "_id",
      foreignField: "customerId",
      as: "orders"
    }
  }
])
```

The fewer documents reach `$lookup`, the fewer joins must be performed.

## Fix 3 - Limit the Joined Documents with Pipeline $lookup

The pipeline form of `$lookup` lets you filter the joined documents before they are returned:

```javascript
db.customers.aggregate([
  { $match: { status: "active" } },
  {
    $lookup: {
      from: "orders",
      let: { custId: "$_id" },
      pipeline: [
        {
          $match: {
            $expr: { $eq: ["$customerId", "$$custId"] },
            status: "shipped"        // filter in the joined collection
          }
        },
        { $project: { orderId: 1, amount: 1, _id: 0 } },  // project early
        { $limit: 10 }
      ],
      as: "recentOrders"
    }
  }
])
```

This pushes the filter and projection into the joined collection, reducing data transferred and processed.

## Fix 4 - Use $unwind Immediately After $lookup

If you only need one matched document per customer, `$unwind` right after `$lookup` allows the query planner to optimize the join:

```javascript
db.customers.aggregate([
  {
    $lookup: {
      from: "subscriptions",
      localField: "_id",
      foreignField: "customerId",
      as: "subscription"
    }
  },
  { $unwind: "$subscription" }   // optimizer can convert to LOOKUP_UNWIND
])
```

MongoDB may fuse these stages into a single efficient operation.

## Checking Execution Plan

```javascript
db.customers.aggregate(
  [{ $lookup: { ... } }],
  { explain: true }
)
```

Look for `EQ_LOOKUP` with `IXSCAN` on the foreign collection. A `COLLSCAN` indicates a missing index.

## Summary

Speed up `$lookup` joins by indexing the foreign collection's join field, adding `$match` before the lookup to reduce input size, and using the pipeline form of `$lookup` with internal filters and projections. Always verify the execution plan to confirm index usage.
