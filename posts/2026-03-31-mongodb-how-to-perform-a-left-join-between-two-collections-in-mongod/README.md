# How to Perform a Left Join Between Two Collections in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, $lookup, Left Join, Aggregation, Join Collections

Description: Learn how to use MongoDB's $lookup aggregation stage to perform a left outer join between two collections, combining related documents from separate collections.

---

## Overview

MongoDB's `$lookup` aggregation stage performs a left outer join between two collections. Every document from the "left" (source) collection is included in the result, with matching documents from the "right" (foreign) collection embedded as an array.

## Basic $lookup Syntax

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",        // collection to join
      localField: "customerId", // field in orders
      foreignField: "_id",      // field in customers
      as: "customer"            // name of the output array field
    }
  }
]);

// Result
// {
//   "_id": "o1",
//   "customerId": "c1",
//   "amount": 100,
//   "customer": [
//     { "_id": "c1", "name": "Alice", "email": "alice@example.com" }
//   ]
// }
```

## Unwinding the Joined Array

`$lookup` returns an array. Use `$unwind` to flatten it to a single embedded document:

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
  {
    $unwind: {
      path: "$customer",
      preserveNullAndEmptyArrays: true  // keep orders with no matching customer
    }
  }
]);
```

## Filtering After the Join

Use `$match` after `$lookup` to filter on joined fields:

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
  { $unwind: { path: "$customer", preserveNullAndEmptyArrays: false } },
  { $match: { "customer.country": "US" } }
]);
```

## $lookup with a Pipeline (Advanced Join)

Use the pipeline form of `$lookup` for more complex join logic including filtering and projection on the joined collection:

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "products",
      let: { orderItems: "$items" },   // variables from the left collection
      pipeline: [
        {
          $match: {
            $expr: {
              $in: ["$sku", "$$orderItems"]
            }
          }
        },
        {
          $project: { sku: 1, name: 1, price: 1, _id: 0 }
        }
      ],
      as: "productDetails"
    }
  }
]);
```

## Projecting Clean Output

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customerInfo"
    }
  },
  { $unwind: { path: "$customerInfo", preserveNullAndEmptyArrays: true } },
  {
    $project: {
      orderId: "$_id",
      amount: 1,
      status: 1,
      customerName: "$customerInfo.name",
      customerEmail: "$customerInfo.email",
      _id: 0
    }
  }
]);
```

## Count of Joined Documents

Use `$size` on the resulting array to count matches without unwinding:

```javascript
db.customers.aggregate([
  {
    $lookup: {
      from: "orders",
      localField: "_id",
      foreignField: "customerId",
      as: "orders"
    }
  },
  {
    $project: {
      name: 1,
      email: 1,
      orderCount: { $size: "$orders" },
      totalSpent: { $sum: "$orders.amount" }
    }
  }
]);
```

## Adding an Index for Join Performance

Always index the `foreignField` to avoid collection scans during the join:

```javascript
// Index on the foreign key field
db.orders.createIndex({ customerId: 1 });

// For the customers collection side
db.customers.createIndex({ _id: 1 });  // _id is indexed by default
```

## Node.js Example

```javascript
async function getOrdersWithCustomers(db, status) {
  return db.collection("orders").aggregate([
    { $match: { status } },
    {
      $lookup: {
        from: "customers",
        localField: "customerId",
        foreignField: "_id",
        as: "customer"
      }
    },
    { $unwind: { path: "$customer", preserveNullAndEmptyArrays: true } },
    {
      $project: {
        amount: 1,
        status: 1,
        createdAt: 1,
        customerName: "$customer.name",
        customerEmail: "$customer.email",
        _id: 1
      }
    }
  ]).toArray();
}
```

## Summary

MongoDB's `$lookup` stage performs a left outer join between collections. The basic form joins on matching field values; the pipeline form supports filtering, projections, and complex expressions on the joined collection. Always use `$unwind` with `preserveNullAndEmptyArrays: true` after `$lookup` to handle unmatched documents gracefully. Index the `foreignField` to avoid slow collection scans during the join, and use `$project` to shape the output into a flat, clean structure.
