# How to Find the Intersection of Two Arrays in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Aggregation

Description: Learn how to find common elements between two arrays in MongoDB using $setIntersection and filter-based aggregation techniques.

---

Finding the intersection of two arrays - elements that appear in both - is a common set operation in MongoDB. Whether comparing two fields within the same document or checking overlap between a document array and a query array, MongoDB provides clean operators for this.

## $setIntersection

`$setIntersection` returns an array of elements that appear in all of the input arrays:

```javascript
db.products.aggregate([
  {
    $project: {
      name: 1,
      commonTags: {
        $setIntersection: ["$tags", ["electronics", "sale", "featured"]]
      }
    }
  }
]);
```

This returns only the tags that appear in both the document's `tags` array and the literal array `["electronics", "sale", "featured"]`.

## Intersection Between Two Document Fields

To compare two array fields within the same document:

```javascript
db.users.aggregate([
  {
    $project: {
      username: 1,
      sharedInterests: {
        $setIntersection: ["$interests", "$followedTopics"]
      }
    }
  }
]);
```

## Filtering to Only Documents with a Non-Empty Intersection

Combine `$setIntersection` with `$match` to find documents that share at least one element with a target array:

```javascript
const targetTags = ["electronics", "computers"];

db.products.aggregate([
  {
    $addFields: {
      matchedTags: { $setIntersection: ["$tags", targetTags] }
    }
  },
  {
    $match: {
      $expr: { $gt: [{ $size: "$matchedTags" }, 0] }
    }
  },
  {
    $project: { matchedTags: 0 }  // remove the temporary field
  }
]);
```

Note: for simple membership queries, `$in` or `$all` on the array field is more efficient and uses indexes. Use `$setIntersection` when you also need the intersection itself in the output.

## Checking Intersection Size

To find documents where the intersection meets a minimum size threshold:

```javascript
db.users.aggregate([
  {
    $addFields: {
      commonCount: {
        $size: {
          $setIntersection: ["$skills", ["python", "mongodb", "redis", "kafka"]]
        }
      }
    }
  },
  {
    $match: { commonCount: { $gte: 2 } }
  },
  {
    $sort: { commonCount: -1 }
  }
]);
```

This finds users who have at least 2 of the specified skills, sorted by most matching.

## Intersection vs. $all

`$all` checks that an array contains all specified values, not the intersection. They solve different problems:

```javascript
// $all: does the array contain ALL of these?
db.products.find({ tags: { $all: ["electronics", "sale"] } });

// $setIntersection: which elements appear in both arrays?
db.products.aggregate([
  { $project: { common: { $setIntersection: ["$tags", ["electronics", "sale"]] } } }
]);
```

Use `$all` when you need a filter with index support. Use `$setIntersection` when you need the actual common elements as output.

## Handling Empty and Null Arrays

`$setIntersection` on a null or missing field produces null. Guard with `$ifNull`:

```javascript
db.products.aggregate([
  {
    $project: {
      commonTags: {
        $setIntersection: [
          { $ifNull: ["$tags", []] },
          ["electronics", "sale"]
        ]
      }
    }
  }
]);
```

## Summary

Use `$setIntersection` in aggregation pipelines when you need the actual common elements between two arrays. Combine it with `$size` and `$match` to filter by intersection size. For simple membership filtering with index support, `$in` or `$all` is the better choice. Always guard against null fields using `$ifNull`.
