# How to Use $lookup for Left Outer Joins in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Join

Description: Learn how to use MongoDB's $lookup stage to perform left outer joins between collections, including simple equality joins and multi-condition pipeline joins.

---

## What Is the $lookup Stage?

The `$lookup` stage performs a left outer join between the current collection (left) and another collection (right) in the same database. For each input document, it adds a new array field containing matching documents from the joined collection. If no matches are found, the array is empty.

## Basic Syntax (Equality Join)

```javascript
db.collection.aggregate([
  {
    $lookup: {
      from: "<foreignCollection>",
      localField: "<fieldInCurrentDoc>",
      foreignField: "<fieldInForeignDoc>",
      as: "<outputArrayField>"
    }
  }
])
```

## Example: Join Orders with Customers

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  }
])
// Each order document gets a "customer" array with matching customer docs
```

## Unwinding the Join Result

Since `$lookup` returns an array, use `$unwind` to flatten it when you expect a single match.

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  { $unwind: { path: "$customer", preserveNullAndEmpty: true } }
])
```

## Pipeline Join (More Flexibility)

The pipeline syntax allows filtering, projecting, and transforming the joined documents.

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "products",
      let: { orderItems: "$items" },
      pipeline: [
        {
          $match: {
            $expr: { $in: ["$_id", "$$orderItems"] }
          }
        },
        { $project: { name: 1, price: 1 } }
      ],
      as: "productDetails"
    }
  }
])
```

## Joining on Multiple Conditions

```javascript
db.shipments.aggregate([
  {
    $lookup: {
      from: "inventory",
      let: { warehouseId: "$warehouseId", sku: "$sku" },
      pipeline: [
        {
          $match: {
            $expr: {
              $and: [
                { $eq: ["$warehouseId", "$$warehouseId"] },
                { $eq: ["$sku", "$$sku"] }
              ]
            }
          }
        }
      ],
      as: "inventoryRecord"
    }
  }
])
```

## Simulating an Inner Join

`$lookup` is a left outer join. To simulate an inner join, filter out documents where the join array is empty.

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  { $match: { customer: { $ne: [] } } }
])
```

## Self-Joins

Join a collection to itself for hierarchical data.

```javascript
db.employees.aggregate([
  {
    $lookup: {
      from: "employees",
      localField: "managerId",
      foreignField: "_id",
      as: "manager"
    }
  }
])
```

## Using Indexes to Optimize $lookup

The `foreignField` should be indexed for efficient joins.

```javascript
// Create index on the join field
db.customers.createIndex({ _id: 1 })

// For pipeline joins, the match in the subpipeline should hit an index
db.products.createIndex({ warehouseId: 1, sku: 1 })
```

## $lookup Limitations

- Works only within the same database
- The joined collection cannot be sharded (in basic equality form; pipeline form allows sharded collections in MongoDB 5.1+)
- Large join arrays can consume significant memory

## Summary

The `$lookup` stage brings SQL-like join capability to MongoDB aggregation. Use the simple equality syntax for straightforward foreign key lookups, and the pipeline syntax for multi-condition joins, filtering, or transforming the joined documents. Always index the foreign join field for optimal performance.
