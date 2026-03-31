# How to Flatten Nested Arrays in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Array, Unwind, Pipeline

Description: Learn how to flatten nested and multi-level arrays in MongoDB aggregation using $unwind, $reduce, and $concatArrays for single and double nesting.

---

Flattening nested arrays is a common aggregation task - normalizing array-of-arrays into a flat array, or using `$unwind` to produce one document per array element. MongoDB provides `$unwind`, `$reduce`, and `$concatArrays` for various flattening scenarios.

## Single-Level Flatten with $unwind

`$unwind` deconstructs an array field, producing one output document per element:

```javascript
// Input document
// { _id: 1, colors: ["red", "green", "blue"] }

db.products.aggregate([
  { $unwind: "$colors" }
])

// Output:
// { _id: 1, colors: "red" }
// { _id: 1, colors: "green" }
// { _id: 1, colors: "blue" }
```

## Flatten and Re-Group

A common pattern: unwind, process, then regroup:

```javascript
db.orders.aggregate([
  { $unwind: "$items" },
  {
    $group: {
      _id: "$items.sku",
      totalQty: { $sum: "$items.qty" },
      revenue: { $sum: { $multiply: ["$items.qty", "$items.price"] } }
    }
  },
  { $sort: { revenue: -1 } }
])
```

## Handling Empty and Null Arrays

By default, `$unwind` removes documents with missing or empty arrays. Preserve them with `preserveNullAndEmptyArrays`:

```javascript
db.products.aggregate([
  {
    $unwind: {
      path: "$variants",
      preserveNullAndEmptyArrays: true
    }
  }
])
```

## Including the Array Index

Track the original array position:

```javascript
db.courses.aggregate([
  {
    $unwind: {
      path: "$lessons",
      includeArrayIndex: "lessonIndex"
    }
  }
])
// Each output document includes "lessonIndex": 0, 1, 2, ...
```

## Flattening Array of Arrays with $reduce

For an array of arrays (nested arrays), `$reduce` combines them:

```javascript
// Document: { _id: 1, nested: [[1, 2], [3, 4], [5]] }
db.data.aggregate([
  {
    $project: {
      flat: {
        $reduce: {
          input: "$nested",
          initialValue: [],
          in: { $concatArrays: ["$$value", "$$this"] }
        }
      }
    }
  }
])
// Result: { _id: 1, flat: [1, 2, 3, 4, 5] }
```

## Flattening to a Specific Depth

For multi-level nesting (array of arrays of arrays), chain the reduce:

```javascript
// Two levels deep: [[["a","b"],["c"]],[["d","e"]]]
db.deep.aggregate([
  {
    $project: {
      level1: {
        $reduce: {
          input: "$nestedDeep",
          initialValue: [],
          in: { $concatArrays: ["$$value", "$$this"] }
        }
      }
    }
  },
  {
    $project: {
      flat: {
        $reduce: {
          input: "$level1",
          initialValue: [],
          in: { $concatArrays: ["$$value", "$$this"] }
        }
      }
    }
  }
])
```

## Extracting a Field From Each Sub-Array Element

When each element is an array of objects and you want one field from each:

```javascript
// Document: { schedules: [ [{day:"Mon",time:"9am"},{day:"Tue",time:"10am"}], [...] ] }
db.staff.aggregate([
  {
    $project: {
      allDays: {
        $reduce: {
          input: "$schedules",
          initialValue: [],
          in: {
            $concatArrays: [
              "$$value",
              {
                $map: {
                  input: "$$this",
                  as: "slot",
                  in: "$$slot.day"
                }
              }
            ]
          }
        }
      }
    }
  }
])
```

## Summary

Use `$unwind` to flatten a single array into multiple documents - one per element. Use `$reduce` with `$concatArrays` to flatten an array of arrays into a single flat array within a document. For two-level nesting, chain two `$reduce` operations or combine `$map` with `$reduce`. Use `preserveNullAndEmptyArrays: true` when documents may have missing or empty arrays that should still appear in the output.
