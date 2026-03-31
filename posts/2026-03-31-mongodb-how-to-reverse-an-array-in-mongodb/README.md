# How to Reverse an Array in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Arrays, Aggregation, $reverseArray

Description: Learn how to reverse array fields in MongoDB using the $reverseArray aggregation operator with practical examples and use cases.

---

## Introduction

MongoDB provides the `$reverseArray` aggregation operator to reverse the elements of an array. This is useful when you want to present data in reverse order, such as showing the most recent items first from a chronologically ordered array, without sorting the entire collection.

## Basic Usage of $reverseArray

Use `$reverseArray` inside a `$project` stage to create a reversed copy of an array field:

```javascript
db.playlists.aggregate([
  {
    $project: {
      name: 1,
      reversedTracks: { $reverseArray: "$tracks" }
    }
  }
]);
```

Given a document like:

```javascript
{ name: "My Playlist", tracks: ["Song A", "Song B", "Song C"] }
```

The output will be:

```javascript
{ name: "My Playlist", reversedTracks: ["Song C", "Song B", "Song A"] }
```

## Reversing an Array In-Place with $set

To update the document's array field with its reversed version:

```javascript
db.playlists.updateMany(
  {},
  [
    {
      $set: {
        tracks: { $reverseArray: "$tracks" }
      }
    }
  ]
);
```

This uses an aggregation pipeline update to apply `$reverseArray` during the update operation.

## Chaining $reverseArray with Other Operators

Reverse an array after filtering or transforming it:

```javascript
db.orders.aggregate([
  {
    $project: {
      recentFirst: {
        $reverseArray: {
          $filter: {
            input: "$items",
            as: "item",
            cond: { $gt: ["$$item.quantity", 0] }
          }
        }
      }
    }
  }
]);
```

## Reversing a Slice of an Array

Combine `$slice` and `$reverseArray` to get the last N elements in reverse:

```javascript
db.events.aggregate([
  {
    $project: {
      lastThreeReversed: {
        $reverseArray: { $slice: ["$events", -3] }
      }
    }
  }
]);
```

## Handling Null and Empty Arrays

`$reverseArray` returns `null` if the input is `null`, and an empty array `[]` if the input is empty:

```javascript
db.test.aggregate([
  {
    $project: {
      reversed: { $reverseArray: "$maybeNull" }
    }
  }
]);
// If maybeNull is null -> reversed: null
// If maybeNull is []   -> reversed: []
```

Guard against nulls using `$ifNull`:

```javascript
db.test.aggregate([
  {
    $project: {
      reversed: {
        $reverseArray: {
          $ifNull: ["$maybeNull", []]
        }
      }
    }
  }
]);
```

## Practical Example - Show Most Recent Log Entries First

```javascript
db.services.aggregate([
  {
    $project: {
      serviceName: 1,
      recentLogs: {
        $reverseArray: { $slice: ["$logs", -10] }
      }
    }
  }
]);
```

## Reversing Nested Arrays

If each document contains nested array documents with their own arrays:

```javascript
db.courses.aggregate([
  {
    $project: {
      title: 1,
      modules: {
        $map: {
          input: "$modules",
          as: "mod",
          in: {
            name: "$$mod.name",
            lessons: { $reverseArray: "$$mod.lessons" }
          }
        }
      }
    }
  }
]);
```

## Summary

The `$reverseArray` operator in MongoDB's aggregation framework makes it straightforward to reverse array fields without additional sorting stages. It works well with `$project`, pipeline updates via `$set`, and can be composed with `$filter`, `$slice`, and `$map` for more complex transformations. Guard against null inputs using `$ifNull` to keep pipelines robust.
