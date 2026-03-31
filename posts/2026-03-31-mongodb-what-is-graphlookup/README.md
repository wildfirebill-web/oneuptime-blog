# What Is $graphLookup and When to Use It in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, graphLookup, Aggregation, Graph, Recursive Query

Description: The $graphLookup stage in MongoDB performs recursive lookups across documents, enabling graph traversals like ancestor chains, org trees, and friend-of-friend queries.

---

## Overview

`$graphLookup` is a MongoDB aggregation pipeline stage that performs a recursive search on a collection, following a path defined by matching field values across documents. It is the MongoDB equivalent of a recursive SQL query or a graph traversal. Unlike `$lookup` which joins two collections once, `$graphLookup` repeatedly follows references until it reaches a specified depth or exhausts all matches.

## When to Use $graphLookup

- Organizational hierarchies (find all reports under a manager)
- Category trees (find all subcategories under a root)
- Friend-of-friend networks (find all connections up to N degrees)
- Bill-of-materials (find all components of a product recursively)
- Route finding and network topology queries

## Syntax

```javascript
{
  $graphLookup: {
    from: <collection>,           // collection to search
    startWith: <expression>,      // field value to start from
    connectFromField: <string>,   // field in current doc to follow
    connectToField: <string>,     // field in target docs to match
    as: <string>,                 // output array field name
    maxDepth: <number>,           // optional: max recursion depth
    depthField: <string>,         // optional: add depth level to results
    restrictSearchWithMatch: <query> // optional: filter traversed docs
  }
}
```

## Example: Organizational Hierarchy

Given an employees collection:

```javascript
db.employees.insertMany([
  { _id: 1, name: "CEO", reportsTo: null },
  { _id: 2, name: "VP Engineering", reportsTo: "CEO" },
  { _id: 3, name: "VP Sales", reportsTo: "CEO" },
  { _id: 4, name: "Engineer Lead", reportsTo: "VP Engineering" },
  { _id: 5, name: "Engineer", reportsTo: "Engineer Lead" }
])
```

Find all direct and indirect reports under "VP Engineering":

```javascript
db.employees.aggregate([
  { $match: { name: "VP Engineering" } },
  {
    $graphLookup: {
      from: "employees",
      startWith: "$name",
      connectFromField: "name",
      connectToField: "reportsTo",
      as: "allReports",
      depthField: "depth"
    }
  },
  {
    $project: {
      name: 1,
      "allReports.name": 1,
      "allReports.depth": 1
    }
  }
])
```

## Example: Find Ancestors

Find the full management chain above an employee:

```javascript
db.employees.aggregate([
  { $match: { name: "Engineer" } },
  {
    $graphLookup: {
      from: "employees",
      startWith: "$reportsTo",
      connectFromField: "reportsTo",
      connectToField: "name",
      as: "managementChain",
      maxDepth: 10
    }
  }
])
```

## Limiting Depth and Filtering

Use `maxDepth` to prevent runaway traversals on large graphs:

```javascript
{
  $graphLookup: {
    from: "categories",
    startWith: "$parentId",
    connectFromField: "parentId",
    connectToField: "_id",
    as: "ancestors",
    maxDepth: 5,
    restrictSearchWithMatch: { active: true }
  }
}
```

## Performance Considerations

- `$graphLookup` loads results into memory. Use `allowDiskUse: true` for large datasets.
- Ensure the `connectToField` is indexed for good traversal performance.
- Set `maxDepth` to avoid full-collection scans on deeply nested graphs.
- The output array can grow large; consider projecting only needed fields.

## Summary

`$graphLookup` enables powerful recursive graph traversals within MongoDB aggregation pipelines. It is the right tool for hierarchical data like org charts, category trees, and social graphs where you need to follow document references across multiple levels in a single query.
