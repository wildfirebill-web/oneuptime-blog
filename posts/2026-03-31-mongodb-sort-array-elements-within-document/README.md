# How to Sort Array Elements Within a Document in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Aggregation

Description: Learn how to sort array elements within a MongoDB document using $sortArray, $push with $sort, and update modifiers.

---

MongoDB's document-level sort tools let you reorder array elements in place - useful for maintaining ordered lists, sorting embedded records by a field, or preparing data for display without fetching and sorting in application code.

## $sortArray in Aggregation (MongoDB 5.2+)

The `$sortArray` operator sorts array elements in a `$project` or `$addFields` stage without modifying the stored document:

```javascript
// Sort a simple array of numbers ascending
db.stats.aggregate([
  {
    $project: {
      sortedScores: {
        $sortArray: {
          input: "$scores",
          sortBy: 1
        }
      }
    }
  }
]);
```

`sortBy: 1` is ascending, `-1` is descending.

## Sorting Arrays of Subdocuments

When array elements are embedded documents, provide a `sortBy` object specifying the sort field:

```javascript
// Sort line items by price descending
db.orders.aggregate([
  {
    $project: {
      orderId: 1,
      sortedItems: {
        $sortArray: {
          input: "$lineItems",
          sortBy: { price: -1 }
        }
      }
    }
  }
]);
```

Multiple sort keys are supported:

```javascript
$sortArray: {
  input: "$lineItems",
  sortBy: { category: 1, price: -1 }
}
```

## Sorting and Persisting with Update + $push $sort

To permanently sort an array as part of an update, use the `$push` update operator with `$sort` and `$slice`:

```javascript
db.orders.updateOne(
  { _id: orderId },
  {
    $push: {
      lineItems: {
        $each: [],      // no new elements to add
        $sort: { price: 1 }
      }
    }
  }
);
```

The `$each: []` with an empty array triggers the sort without adding any elements. This is the idiomatic way to re-sort an existing array in place.

## Maintaining a Sorted Array on Insert

When adding new elements and keeping the array sorted, combine `$push`, `$each`, and `$sort`:

```javascript
db.leaderboards.updateOne(
  { gameId: "GAME-001" },
  {
    $push: {
      scores: {
        $each: [{ player: "Alice", score: 4200 }],
        $sort: { score: -1 },
        $slice: 10   // keep only the top 10
      }
    }
  }
);
```

This inserts the new score, sorts all scores descending, and trims to the top 10 in a single atomic operation.

## Sorting String Arrays

For sorting arrays of strings, `$sortArray` with `sortBy: 1` applies lexicographic ordering:

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      sortedTags: {
        $sortArray: {
          input: "$tags",
          sortBy: 1
        }
      }
    }
  }
]);
```

## Pre-5.2 Alternative: $unwind, $sort, $group

On older MongoDB versions without `$sortArray`, unwind, sort, and regroup:

```javascript
db.orders.aggregate([
  { $unwind: "$lineItems" },
  { $sort: { "lineItems.price": 1 } },
  {
    $group: {
      _id: "$_id",
      orderId: { $first: "$orderId" },
      lineItems: { $push: "$lineItems" }
    }
  }
]);
```

This approach is more verbose and creates one document per element before regrouping, so it is less efficient than `$sortArray` on large arrays.

## Summary

Use `$sortArray` (MongoDB 5.2+) for in-pipeline sorting without modifying stored data. Use `$push` with `$sort` and `$each: []` for in-place array sorting in update operations. Combine `$push`, `$sort`, and `$slice` to maintain a capped sorted list like a leaderboard.
