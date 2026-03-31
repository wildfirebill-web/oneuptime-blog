# How to Use $graphLookup for Recursive Graph Traversal in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Aggregation, Pipeline Stage, Graph

Description: Learn how MongoDB's $graphLookup performs recursive traversal of graph-like data structures such as org charts, category trees, and social networks within aggregation pipelines.

---

## What Is the $graphLookup Stage?

The `$graphLookup` stage performs a recursive lookup through a collection, following a chain of references to traverse graph-like data. It is designed for hierarchical structures like organizational charts, product category trees, social graphs, and network topologies - data patterns where you need to follow relationships to arbitrary depth.

## Basic Syntax

```javascript
db.collection.aggregate([
  {
    $graphLookup: {
      from: "<collection>",
      startWith: "<expression>",
      connectFromField: "<fieldName>",
      connectToField: "<fieldName>",
      as: "<outputArrayField>",
      maxDepth: <number>,
      depthField: "<fieldName>",
      restrictSearchWithMatch: { <query> }
    }
  }
])
```

## Example: Finding All Reports in an Org Chart

```javascript
// Collection: employees
// { _id: "emp-1", name: "Alice", managerId: null }
// { _id: "emp-2", name: "Bob", managerId: "emp-1" }
// { _id: "emp-3", name: "Carol", managerId: "emp-1" }
// { _id: "emp-4", name: "Dave", managerId: "emp-2" }

db.employees.aggregate([
  { $match: { name: "Alice" } },
  {
    $graphLookup: {
      from: "employees",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "managerId",
      as: "directAndIndirectReports"
    }
  }
])
// Returns Alice's full reporting chain: Bob, Carol, Dave
```

## Example: Finding All Ancestors (Parent Chain)

```javascript
db.categories.aggregate([
  { $match: { name: "Smartphones" } },
  {
    $graphLookup: {
      from: "categories",
      startWith: "$parentId",
      connectFromField: "parentId",
      connectToField: "_id",
      as: "ancestors"
    }
  }
])
// Returns: Electronics -> Consumer Electronics -> Smartphones (ancestors of Smartphones)
```

## Limiting Depth

Use `maxDepth` to prevent unbounded traversal in large graphs.

```javascript
db.employees.aggregate([
  { $match: { _id: "emp-1" } },
  {
    $graphLookup: {
      from: "employees",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "managerId",
      maxDepth: 2,
      as: "twoLevelsDown"
    }
  }
])
```

## Tracking Traversal Depth

Add `depthField` to know how many hops each result is from the start.

```javascript
db.employees.aggregate([
  { $match: { _id: "emp-1" } },
  {
    $graphLookup: {
      from: "employees",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "managerId",
      depthField: "level",
      as: "orgTree"
    }
  }
])
// Each employee in orgTree has a "level" field: 0=direct reports, 1=two levels down...
```

## Filtering During Traversal

Use `restrictSearchWithMatch` to only traverse nodes matching a condition.

```javascript
db.employees.aggregate([
  { $match: { _id: "emp-1" } },
  {
    $graphLookup: {
      from: "employees",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "managerId",
      restrictSearchWithMatch: { active: true },
      as: "activeReports"
    }
  }
])
// Only traverses through active employees
```

## Social Network Friend Connections

```javascript
db.users.aggregate([
  { $match: { _id: "user-alice" } },
  {
    $graphLookup: {
      from: "users",
      startWith: "$friends",
      connectFromField: "friends",
      connectToField: "_id",
      maxDepth: 2,
      depthField: "connectionLevel",
      as: "network"
    }
  }
])
// Returns friends (depth 1) and friends-of-friends (depth 2)
```

## Performance Notes

- Index the `connectToField` for efficient traversal
- Use `maxDepth` to prevent performance issues on large graphs
- `$graphLookup` is memory-intensive; use `restrictSearchWithMatch` to prune early

```javascript
db.employees.createIndex({ managerId: 1 })
```

## Summary

The `$graphLookup` stage enables recursive graph traversal directly in MongoDB aggregation pipelines. It handles hierarchical and network data by following document references to arbitrary depth. Use `maxDepth` to control traversal scope, `depthField` to track hop count, and `restrictSearchWithMatch` to prune the search space for better performance.
