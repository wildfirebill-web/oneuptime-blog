# How to Update an Element in a Nested Array in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Update, Array, Positional Operator, Operator

Description: Learn how to update a specific element inside a nested array in MongoDB using the positional operator, $[identifier] arrayFilters, and aggregation pipeline updates.

---

Updating elements inside nested arrays is one of MongoDB's more nuanced operations. The right approach depends on whether you know the array index, whether you're matching by element value, or whether you need to update multiple matching elements.

## Sample Document

```javascript
db.orders.insertOne({
  _id: 1,
  customerId: "cust-42",
  items: [
    { sku: "A100", qty: 2, status: "pending" },
    { sku: "B200", qty: 1, status: "pending" },
    { sku: "C300", qty: 3, status: "shipped" }
  ]
})
```

## The $ Positional Operator (First Match)

The `$` positional operator updates the first array element that matches the query condition:

```javascript
// Update the status of the item with sku "B200"
db.orders.updateOne(
  { _id: 1, "items.sku": "B200" },
  { $set: { "items.$.status": "shipped" } }
)
```

The `$` in the update path refers to the index of the matched element.

## Updating Multiple Fields of the Matched Element

```javascript
db.orders.updateOne(
  { _id: 1, "items.sku": "A100" },
  {
    $set: {
      "items.$.status": "shipped",
      "items.$.shippedAt": new Date()
    }
  }
)
```

## The $[identifier] Filtered Positional Operator

Use `arrayFilters` to update all array elements matching a condition (not just the first):

```javascript
// Update all "pending" items to "processing"
db.orders.updateOne(
  { _id: 1 },
  { $set: { "items.$[item].status": "processing" } },
  {
    arrayFilters: [{ "item.status": "pending" }]
  }
)
```

## Updating Elements in Nested Arrays (Array of Arrays)

For doubly nested arrays, chain the positional operators:

```javascript
db.courses.updateOne(
  {
    _id: courseId,
    "modules.lessons.lessonId": "lesson-5"
  },
  {
    $set: { "modules.$[].lessons.$[lesson].completed": true }
  },
  {
    arrayFilters: [{ "lesson.lessonId": "lesson-5" }]
  }
)
```

## Updating by Array Index (Known Position)

If you know the index, use it directly:

```javascript
// Update the second item (index 1)
db.orders.updateOne(
  { _id: 1 },
  { $set: { "items.1.status": "shipped" } }
)
```

This is brittle if the array order can change - prefer matching by a field value.

## Aggregation Pipeline Update (MongoDB 4.2+)

For complex transformations, use an aggregation pipeline in the update:

```javascript
db.orders.updateOne(
  { _id: 1 },
  [
    {
      $set: {
        items: {
          $map: {
            input: "$items",
            as: "item",
            in: {
              $mergeObjects: [
                "$$item",
                {
                  status: {
                    $cond: {
                      if: { $eq: ["$$item.sku", "A100"] },
                      then: "delivered",
                      else: "$$item.status"
                    }
                  }
                }
              ]
            }
          }
        }
      }
    }
  ]
)
```

## Incrementing a Numeric Field in an Array Element

```javascript
// Add 5 to qty for the item with sku "B200"
db.orders.updateOne(
  { _id: 1, "items.sku": "B200" },
  { $inc: { "items.$.qty": 5 } }
)
```

## Summary

Use the `$` positional operator to update the first array element matching the query filter. Use `$[identifier]` with `arrayFilters` to update all elements matching a condition in a single operation. For complex transformations like conditional updates based on current values, use an aggregation pipeline update. Always prefer matching by a field value over using a hardcoded index position.
