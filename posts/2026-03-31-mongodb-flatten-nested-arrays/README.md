# How to Flatten Nested Arrays in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Array, Aggregation

Description: Learn how to flatten nested arrays in MongoDB using $unwind, $reduce, and $concatArrays to produce flat lists from deeply nested structures.

---

Nested arrays - arrays of arrays - appear frequently when documents store multi-level hierarchies. Flattening them into a single-level list is often necessary for aggregation, querying individual elements, or preparing data for export.

## One Level of Nesting with $unwind

`$unwind` deconstructs an array field into separate documents, one per element. Running `$unwind` twice on nested arrays flattens one level:

```javascript
// Documents like: { category: "tools", subgroups: [["hammer", "wrench"], ["drill", "saw"]] }
db.catalog.aggregate([
  { $unwind: "$subgroups" },     // subgroups becomes each inner array
  { $unwind: "$subgroups" }      // subgroups becomes each string element
]);
```

The first `$unwind` makes `subgroups` the inner array. The second makes it a scalar. The result is one document per leaf value.

## Flattening with $reduce and $concatArrays

For a more functional approach that keeps one document per input document, use `$reduce` to merge all inner arrays into one:

```javascript
db.catalog.aggregate([
  {
    $project: {
      flatItems: {
        $reduce: {
          input: "$subgroups",
          initialValue: [],
          in: { $concatArrays: ["$$value", "$$this"] }
        }
      }
    }
  }
]);
```

This produces a single document with `flatItems` containing all leaf strings, rather than multiplying documents with `$unwind`.

## Handling Deeper Nesting

For two levels of nesting (arrays of arrays of arrays), extend the `$reduce` approach:

```javascript
// Documents: { matrix: [[[1,2],[3,4]], [[5,6],[7,8]]] }
db.matrices.aggregate([
  {
    $project: {
      // First flatten outer -> middle
      middle: {
        $reduce: {
          input: "$matrix",
          initialValue: [],
          in: { $concatArrays: ["$$value", "$$this"] }
        }
      }
    }
  },
  {
    $project: {
      // Then flatten middle -> flat
      flat: {
        $reduce: {
          input: "$middle",
          initialValue: [],
          in: { $concatArrays: ["$$value", "$$this"] }
        }
      }
    }
  }
]);
```

Chain as many `$project` stages as there are nesting levels.

## Using $unwind with preserveNullAndEmptyArrays

When documents may have missing or empty nested arrays, prevent `$unwind` from dropping those documents:

```javascript
db.catalog.aggregate([
  {
    $unwind: {
      path: "$subgroups",
      preserveNullAndEmptyArrays: true
    }
  }
]);
```

## Flattening to Re-group

A common pattern is to flatten, process, and re-group. For example, collecting all tags from a nested tag hierarchy and counting unique ones:

```javascript
db.articles.aggregate([
  {
    $project: {
      allTags: {
        $reduce: {
          input: "$tagGroups",
          initialValue: [],
          in: { $concatArrays: ["$$value", "$$this"] }
        }
      }
    }
  },
  { $unwind: "$allTags" },
  {
    $group: {
      _id: "$allTags",
      count: { $sum: 1 }
    }
  },
  { $sort: { count: -1 } }
]);
```

## Summary

Use `$unwind` twice to flatten one level of array nesting while multiplying documents. Use `$reduce` with `$concatArrays` to flatten within a single document while preserving the one-document-in, one-document-out structure. For deep nesting, chain multiple `$reduce` stages. Choose based on whether you need one output document per input document or one per leaf element.
