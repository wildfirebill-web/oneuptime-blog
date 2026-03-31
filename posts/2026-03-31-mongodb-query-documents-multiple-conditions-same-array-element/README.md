# How to Query Documents with Multiple Conditions on the Same Array Element in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Query, Array, Operator, Aggregation

Description: Learn how to use $elemMatch in MongoDB to query documents where a single array element satisfies multiple conditions simultaneously.

---

When querying arrays in MongoDB, a common mistake is applying multiple conditions that MongoDB evaluates independently across different array elements rather than against the same element. Understanding the difference between dot-notation queries and `$elemMatch` is essential for correct array filtering.

## The Problem with Naive Array Queries

Consider an `orders` collection where each document has a `lineItems` array:

```json
{
  "_id": 1,
  "lineItems": [
    { "product": "Widget", "qty": 5, "price": 10 },
    { "product": "Gadget", "qty": 2, "price": 50 }
  ]
}
```

If you run:

```javascript
db.orders.find({
  "lineItems.qty": { $gt: 3 },
  "lineItems.price": { $gt: 40 }
})
```

MongoDB returns this document because `qty > 3` matches the first element and `price > 40` matches the second element - not the same element. This is often not what you want.

## Using $elemMatch for Element-Level Conditions

`$elemMatch` forces all conditions to be evaluated against the same array element:

```javascript
db.orders.find({
  lineItems: {
    $elemMatch: {
      qty: { $gt: 3 },
      price: { $gt: 40 }
    }
  }
})
```

This query only returns documents where a single `lineItems` element has both `qty > 3` AND `price > 40`. With the sample data above, no document matches because no single item satisfies both conditions.

## Practical Example with Real Data

```javascript
// Insert sample inventory
db.inventory.insertMany([
  {
    warehouse: "A",
    items: [
      { sku: "abc", inStock: true, qty: 100 },
      { sku: "def", inStock: false, qty: 0 }
    ]
  },
  {
    warehouse: "B",
    items: [
      { sku: "abc", inStock: false, qty: 50 },
      { sku: "def", inStock: true, qty: 200 }
    ]
  }
])

// Find warehouses that have 'abc' both in stock AND with qty > 60
db.inventory.find({
  items: {
    $elemMatch: {
      sku: "abc",
      inStock: true,
      qty: { $gt: 60 }
    }
  }
})
// Returns warehouse A only
```

## Using $elemMatch in Aggregation Pipelines

You can also use `$elemMatch` inside `$match` stages:

```javascript
db.inventory.aggregate([
  {
    $match: {
      items: {
        $elemMatch: {
          inStock: true,
          qty: { $gte: 100 }
        }
      }
    }
  },
  {
    $project: {
      warehouse: 1,
      highStockItems: {
        $filter: {
          input: "$items",
          as: "item",
          cond: {
            $and: [
              { $eq: ["$$item.inStock", true] },
              { $gte: ["$$item.qty", 100] }
            ]
          }
        }
      }
    }
  }
])
```

## When $elemMatch Is Not Needed

For single-field conditions, `$elemMatch` is unnecessary:

```javascript
// These are equivalent for single conditions
db.inventory.find({ "items.qty": { $gt: 50 } })
db.inventory.find({ items: { $elemMatch: { qty: { $gt: 50 } } } })
```

`$elemMatch` also applies in projections to return only the first matching array element rather than the full array.

## Index Considerations

A compound multikey index on array fields works well with `$elemMatch` queries:

```javascript
db.inventory.createIndex({ "items.sku": 1, "items.qty": 1 })
```

Run `explain()` to confirm index usage, as multikey index behavior with `$elemMatch` depends on how conditions align with index keys.

## Summary

Use `$elemMatch` whenever you need multiple conditions to apply to the same array element. Without it, MongoDB evaluates conditions across any combination of array elements, which can produce incorrect results. This is one of the most common sources of subtle query bugs in MongoDB applications.
