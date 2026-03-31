# How to Use $setIntersection and $setDifference in MongoDB Aggregation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Array, $setIntersection, $setDifference

Description: Learn how to use $setIntersection and $setDifference in MongoDB aggregation to find common elements between arrays and compute array differences in documents.

---

MongoDB's set operators treat arrays as mathematical sets - ignoring duplicates and order. `$setIntersection` finds elements common to all input arrays, and `$setDifference` returns elements in the first array that are not in the second.

## $setIntersection

Returns a de-duplicated array of elements that appear in every input array:

```js
{ $setIntersection: [ <array1>, <array2>, ... ] }
```

### Finding Common Tags

```js
db.articles.aggregate([
  {
    $project: {
      title: 1,
      sharedTags: {
        $setIntersection: ["$userTags", "$editorTags"]
      }
    }
  }
]);
```

If a document has `userTags: ["nodejs","docker","mongodb"]` and `editorTags: ["mongodb","kubernetes","docker"]`, the result is `sharedTags: ["docker","mongodb"]` (order not guaranteed).

### Checking for Overlap

Combine with `$size` to check whether two arrays share any elements:

```js
db.products.aggregate([
  {
    $project: {
      name: 1,
      hasCommonCategory: {
        $gt: [
          { $size: { $setIntersection: ["$categories", ["electronics", "gadgets"]] } },
          0
        ]
      }
    }
  }
]);
```

### Multi-Array Intersection

Pass more than two arrays to find elements common to all of them:

```js
db.reports.aggregate([
  {
    $project: {
      commonRegions: {
        $setIntersection: ["$q1Regions", "$q2Regions", "$q3Regions", "$q4Regions"]
      }
    }
  }
]);
```

## $setDifference

Returns elements in the first array that are not in the second:

```js
{ $setDifference: [ <array1>, <array2> ] }
```

### Finding Removed Tags

```js
db.articles.aggregate([
  {
    $project: {
      title: 1,
      removedTags: {
        $setDifference: ["$oldTags", "$newTags"]
      },
      addedTags: {
        $setDifference: ["$newTags", "$oldTags"]
      }
    }
  }
]);
```

### Users Without a Specific Permission

Find which permissions a role has that are not in the "read-only" baseline:

```js
db.roles.aggregate([
  {
    $project: {
      roleName: 1,
      extraPermissions: {
        $setDifference: [
          "$permissions",
          ["read:own", "list:own"]
        ]
      }
    }
  },
  {
    $match: {
      $expr: { $gt: [{ $size: "$extraPermissions" }, 0] }
    }
  }
]);
```

## Combining Both Operators

Compute a symmetric difference (elements in either array but not both):

```js
db.documents.aggregate([
  {
    $project: {
      symmetricDiff: {
        $concatArrays: [
          { $setDifference: ["$arrayA", "$arrayB"] },
          { $setDifference: ["$arrayB", "$arrayA"] }
        ]
      }
    }
  }
]);
```

## Null Handling

If either input is `null` or missing, the result is `null`. Use `$ifNull` to default to empty arrays:

```js
db.users.aggregate([
  {
    $project: {
      newFeatures: {
        $setDifference: [
          { $ifNull: ["$enabledFeatures", []] },
          { $ifNull: ["$seenFeatures", []] }
        ]
      }
    }
  }
]);
```

## Summary

`$setIntersection` and `$setDifference` bring set algebra to MongoDB aggregation pipelines. Use `$setIntersection` to find common array elements across fields or documents, and `$setDifference` to compute what was added or removed between two arrays. Both operators deduplicate their inputs automatically, making them safe to use with arrays that may contain repeated values.
