# How to Use $elemMatch for Array Projection in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Projection

Description: Learn how to use $elemMatch in MongoDB projections to return only the first array element matching specific criteria, independent of your query filter.

---

MongoDB's `$elemMatch` operator serves two distinct roles: as a query operator to match documents where any array element meets multiple conditions, and as a **projection operator** to limit the returned array to the first element satisfying given criteria. This post focuses on the projection use case.

## Projection vs. Query $elemMatch

When used in the projection part of a `find()` call, `$elemMatch` lets you specify match conditions independently of the query filter. This is the key difference from the `$` positional operator, which reuses the query condition.

## Basic Example

Suppose you have an `orders` collection:

```javascript
db.orders.insertMany([
  {
    customer: "Alice",
    items: [
      { sku: "A1", qty: 3, price: 10 },
      { sku: "B2", qty: 1, price: 50 },
      { sku: "C3", qty: 2, price: 20 }
    ]
  },
  {
    customer: "Bob",
    items: [
      { sku: "A1", qty: 5, price: 10 },
      { sku: "D4", qty: 1, price: 75 }
    ]
  }
])
```

To retrieve each order but include only the first item where `qty` is greater than 2 and `price` is less than 25:

```javascript
db.orders.find(
  {},
  {
    customer: 1,
    items: { $elemMatch: { qty: { $gt: 2 }, price: { $lt: 25 } } }
  }
)
```

Result:

```javascript
[
  { customer: "Alice", items: [{ sku: "A1", qty: 3, price: 10 }] },
  { customer: "Bob",   items: [{ sku: "A1", qty: 5, price: 10 }] }
]
```

Documents where no array element satisfies the conditions will simply omit the `items` field from the result.

## Using $elemMatch When Query Filters on Other Fields

A common scenario is querying by one field but projecting based on array criteria:

```javascript
db.orders.find(
  { customer: "Alice" },
  { items: { $elemMatch: { sku: "B2" } } }
)
```

This finds Alice's order and returns only the item with sku `B2`. Without `$elemMatch`, the entire `items` array would be returned.

## Difference from the $ Positional Operator

```javascript
// $ operator: projection condition must match the query condition
db.orders.find(
  { "items.sku": "A1" },
  { "items.$": 1 }
)

// $elemMatch in projection: independent criteria, not tied to query
db.orders.find(
  { customer: "Alice" },
  { items: { $elemMatch: { sku: "A1" } } }
)
```

Use `$elemMatch` projection when:
- Your query does not filter on the array field at all.
- You need multi-condition matching on the array subdocument in the projection.
- You want to project based on different criteria than the query filter.

## Only the First Match is Returned

Like the `$` operator, `$elemMatch` in projections returns only the **first** matching array element. If you need all matching elements, use the aggregation pipeline with `$filter`.

```javascript
// Aggregation approach to get ALL matching elements
db.orders.aggregate([
  { $match: { customer: "Alice" } },
  {
    $project: {
      customer: 1,
      items: {
        $filter: {
          input: "$items",
          as: "item",
          cond: { $and: [
            { $gt: ["$$item.qty", 2] },
            { $lt: ["$$item.price", 25] }
          ]}
        }
      }
    }
  }
])
```

## Restrictions

- Only one `$elemMatch` projection per array field is supported.
- It cannot be combined with the `$` positional operator on the same field.
- It returns at most one element per document.

## Summary

The `$elemMatch` projection operator in MongoDB lets you return only the first array element matching a set of conditions, keeping responses lean when you only need a specific subdocument. Unlike the `$` positional operator, its criteria are independent of the query filter. Use the aggregation `$filter` stage when you need all matching elements rather than just the first one.
