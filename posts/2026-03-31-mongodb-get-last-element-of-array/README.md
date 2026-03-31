# How to Get the Last Element of an Array in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Aggregation

Description: Learn how to retrieve the last element of an array in MongoDB using $arrayElemAt with -1, $last, and $slice operators.

---

Getting the last element of an array in MongoDB - the most recent event, the latest status, the final item - is as common as getting the first. MongoDB provides clean operators for this without needing to know the array length in advance.

## $arrayElemAt with Index -1

In an aggregation pipeline, `$arrayElemAt` with index `-1` returns the last element of the array:

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      latestTag: { $arrayElemAt: ["$tags", -1] }
    }
  }
]);
```

Negative indices count from the end: `-1` is the last element, `-2` is second to last, and so on. This mirrors Python's list indexing convention.

## $last as Array Operator (MongoDB 4.4+)

MongoDB 4.4 introduced `$last` as an expression operator usable in `$project` and `$addFields`:

```javascript
db.shipments.aggregate([
  {
    $project: {
      orderId: 1,
      finalStatus: { $last: "$statusHistory" }
    }
  }
]);
```

This is equivalent to `{ $arrayElemAt: ["$statusHistory", -1] }` and is more readable when the intent is "get the last item."

## $last Accumulator in $group

In a `$group` stage, `$last` returns the last value encountered in the group:

```javascript
db.events.aggregate([
  { $sort: { timestamp: 1 } },
  {
    $group: {
      _id: "$userId",
      lastEvent: { $last: "$eventType" },
      lastEventTime: { $last: "$timestamp" }
    }
  }
]);
```

Sort before grouping to make "last" deterministic. Without a sort, MongoDB processes documents in natural order, which is not guaranteed to be stable.

## Using $slice with -1

The `$slice` aggregation operator can extract the last N elements by passing a negative count:

```javascript
// Get the last element as a single-element array
db.products.aggregate([
  {
    $project: {
      lastTag: { $slice: ["$tags", -1] }
    }
  }
]);
```

Note that `$slice` returns an array, while `$arrayElemAt` and `$last` return a scalar. Use `$slice` when you want the last N elements as an array, and `$arrayElemAt[-1]` when you want a single scalar value.

In a `find` projection:

```javascript
// Project only the last tag in a find query
db.products.find({}, { tags: { $slice: -1 }, name: 1 });
```

## Handling Empty Arrays

When the array might be empty, `$arrayElemAt` returns null. Combine with `$ifNull` to provide a fallback:

```javascript
db.shipments.aggregate([
  {
    $project: {
      finalStatus: {
        $ifNull: [
          { $arrayElemAt: ["$statusHistory", -1] },
          "unknown"
        ]
      }
    }
  }
]);
```

## Practical Pattern: Most Recent Entry

A common pattern is to maintain an array of timestamped events and retrieve the most recent one:

```javascript
db.devices.aggregate([
  {
    $project: {
      deviceId: 1,
      latestReading: { $last: "$readings" },
      readingCount: { $size: "$readings" }
    }
  },
  {
    $match: {
      "latestReading.value": { $gt: 100 }
    }
  }
]);
```

This retrieves the last reading from each device's readings array and filters by value - all in one pipeline pass.

## Summary

Use `$arrayElemAt` with index `-1` or the `$last` expression operator (MongoDB 4.4+) to get the last element of an array as a scalar in aggregation. Use `$slice: -1` in `find` projections or aggregation when you need the result as a single-element array. Always sort before using `$last` in a `$group` stage.
