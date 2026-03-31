# How to Join More Than Two Collections in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $lookup, Multiple Joins, Aggregation, Collections

Description: Learn how to chain multiple $lookup stages in a MongoDB aggregation pipeline to join three or more collections in a single query.

---

## Overview

MongoDB's aggregation pipeline allows multiple `$lookup` stages, making it possible to join three or more collections in a single pipeline. Each `$lookup` stage adds a new array field to the documents, which can then be used in subsequent stages.

## Sample Collections

```javascript
// orders collection
{ "_id": "o1", "customerId": "c1", "productId": "p1", "amount": 100 }

// customers collection
{ "_id": "c1", "name": "Alice", "regionId": "r1" }

// products collection
{ "_id": "p1", "name": "Laptop", "categoryId": "cat1" }

// regions collection
{ "_id": "r1", "name": "North America" }
```

## Chaining Multiple $lookup Stages

```javascript
db.orders.aggregate([
  // Join with customers
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  { $unwind: { path: "$customer", preserveNullAndEmptyArrays: true } },

  // Join with products
  {
    $lookup: {
      from: "products",
      localField: "productId",
      foreignField: "_id",
      as: "product"
    }
  },
  { $unwind: { path: "$product", preserveNullAndEmptyArrays: true } },

  // Join regions using a field from the already-joined customer
  {
    $lookup: {
      from: "regions",
      localField: "customer.regionId",
      foreignField: "_id",
      as: "region"
    }
  },
  { $unwind: { path: "$region", preserveNullAndEmptyArrays: true } },

  // Project clean output
  {
    $project: {
      orderId: "$_id",
      amount: 1,
      customerName: "$customer.name",
      productName: "$product.name",
      region: "$region.name",
      _id: 0
    }
  }
]);
```

## Joining Through a Junction Collection (Many-to-Many)

For many-to-many relationships via a junction table:

```javascript
// orderItems collection (junction)
{ "_id": "oi1", "orderId": "o1", "productId": "p1", "qty": 2 }
{ "_id": "oi2", "orderId": "o1", "productId": "p2", "qty": 1 }

db.orders.aggregate([
  // Join order items
  {
    $lookup: {
      from: "orderItems",
      localField: "_id",
      foreignField: "orderId",
      as: "items"
    }
  },
  // Unwind items to join each item with its product
  { $unwind: { path: "$items", preserveNullAndEmptyArrays: true } },
  // Join products
  {
    $lookup: {
      from: "products",
      localField: "items.productId",
      foreignField: "_id",
      as: "items.product"
    }
  },
  { $unwind: { path: "$items.product", preserveNullAndEmptyArrays: true } },
  // Regroup items back per order
  {
    $group: {
      _id: "$_id",
      customerId: { $first: "$customerId" },
      items: {
        $push: {
          qty: "$items.qty",
          productName: "$items.product.name",
          productPrice: "$items.product.price"
        }
      }
    }
  }
]);
```

## Using Pipeline Form for Complex Join Conditions

Chain pipeline-form `$lookup` when you need filtering on the joined collection:

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      let: { custId: "$customerId" },
      pipeline: [
        { $match: { $expr: { $eq: ["$_id", "$$custId"] } } },
        { $project: { name: 1, email: 1, tier: 1, _id: 0 } }
      ],
      as: "customer"
    }
  },
  { $unwind: "$customer" },
  {
    $lookup: {
      from: "products",
      let: { prodId: "$productId", custTier: "$customer.tier" },
      pipeline: [
        { $match: { $expr: { $eq: ["$_id", "$$prodId"] } } },
        {
          $project: {
            name: 1,
            price: 1,
            discountForTier: {
              $cond: [{ $eq: ["$$custTier", "premium"] }, "$premiumDiscount", 0]
            }
          }
        }
      ],
      as: "product"
    }
  },
  { $unwind: "$product" }
]);
```

## Performance Considerations

- Add indexes on all `foreignField` values to avoid collection scans.
- Use `$match` as early as possible to reduce the number of documents entering each `$lookup`.
- Use `$project` between `$lookup` stages to remove fields you do not need before the next join.

```javascript
db.customers.createIndex({ _id: 1 });   // already indexed
db.products.createIndex({ _id: 1 });    // already indexed
db.regions.createIndex({ _id: 1 });     // already indexed
db.orderItems.createIndex({ orderId: 1 });
db.orderItems.createIndex({ productId: 1 });
```

## Summary

MongoDB supports joining more than two collections by chaining multiple `$lookup` stages in a single aggregation pipeline. After each `$lookup`, use `$unwind` with `preserveNullAndEmptyArrays: true` to flatten the result before the next join. Place `$match` stages as early as possible to minimize document count, and use `$project` between stages to drop unnecessary fields. Always index all fields used in `localField`/`foreignField` pairs for optimal performance.
