# How to Create a Standard View in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, View, Aggregation, Database Design, Query

Description: Learn how to create a standard read-only view in MongoDB using createView so you can simplify queries and restrict field access without duplicating data.

---

A MongoDB view is a named, read-only query that behaves like a collection. It is defined over an aggregation pipeline and stores no data of its own - every query against a view re-executes the pipeline against the source collection.

## Creating a View

Use `db.createView()` or the `create` command:

```javascript
db.createView(
  "activeUsers",          // view name
  "users",                // source collection
  [                       // aggregation pipeline
    { $match: { status: "active" } },
    { $project: { name: 1, email: 1, createdAt: 1, _id: 0 } }
  ]
)
```

The view `activeUsers` now exists in the same database and can be queried like any collection.

## Querying a View

```javascript
db.activeUsers.find({ email: /gmail/ })
db.activeUsers.countDocuments()
db.activeUsers.find().sort({ createdAt: -1 }).limit(10)
```

All standard `find`, `countDocuments`, and aggregation operations work. Write operations are not permitted.

## Creating a View with the create Command

```javascript
db.runCommand({
  create: "orderSummaries",
  viewOn: "orders",
  pipeline: [
    { $match: { status: { $in: ["shipped", "delivered"] } } },
    {
      $project: {
        orderId: "$_id",
        customerId: 1,
        total: 1,
        status: 1,
        _id: 0
      }
    }
  ]
})
```

## Listing Views

Views appear in `listCollections` output with `type: "view"`:

```javascript
db.getCollectionInfos({ type: "view" })
```

Or in mongosh:

```javascript
show collections
```

Views are listed alongside regular collections.

## Joining Collections in a View

Views can use `$lookup` to combine data from multiple collections:

```javascript
db.createView("ordersWithCustomers", "orders", [
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  { $unwind: "$customer" },
  {
    $project: {
      orderId: "$_id",
      total: 1,
      "customer.name": 1,
      "customer.email": 1,
      _id: 0
    }
  }
])
```

## Security Use Case

Create a view that excludes sensitive fields for less privileged roles:

```javascript
db.createView("usersPublic", "users", [
  { $project: { passwordHash: 0, ssn: 0, paymentInfo: 0 } }
])
```

Grant read access on `usersPublic` instead of the raw `users` collection.

## Summary

MongoDB standard views are created with `db.createView()` and wrap an aggregation pipeline over a source collection. They are read-only, store no data, and appear alongside regular collections. Use views to simplify complex queries, restrict field visibility, or present joined data without schema duplication.
