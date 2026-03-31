# How to Implement the Tree Pattern in MongoDB (Parent References)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Tree Pattern, Parent Reference, Schema Design, Hierarchical Data

Description: Learn how to model hierarchical tree structures in MongoDB using the Parent References pattern for efficient ancestor lookups and tree traversal.

---

## What Is the Parent References Pattern?

The Parent References pattern is one of the most straightforward ways to represent tree structures in MongoDB. Each document stores a reference to its direct parent, creating a linked chain from leaf nodes up to the root.

This pattern is ideal when you frequently need to find the parent of a node, traverse upward through the hierarchy, or update a node without touching its children.

## When to Use Parent References

Use this pattern when:
- You need to move subtrees frequently (only one field update per node)
- Your application queries a node's direct parent often
- The tree depth is unpredictable and can grow large
- You want simple inserts without reconstructing full paths

Avoid this pattern when you need to retrieve an entire subtree in a single query, since that requires multiple round-trips.

## Document Structure

Each document holds a `parent` field pointing to the `_id` of its parent. The root node has `parent: null`.

```javascript
// Category tree example
db.categories.insertMany([
  { _id: 1, name: "Electronics", parent: null },
  { _id: 2, name: "Computers", parent: 1 },
  { _id: 3, name: "Laptops", parent: 2 },
  { _id: 4, name: "Gaming Laptops", parent: 3 },
  { _id: 5, name: "Phones", parent: 1 },
  { _id: 6, name: "Smartphones", parent: 5 }
]);
```

## Finding Direct Children

To find all immediate children of a node, query by the parent field:

```javascript
// Get all direct children of "Electronics" (id: 1)
db.categories.find({ parent: 1 });
// Returns: Computers, Phones
```

## Finding the Parent of a Node

To traverse upward, look up the document and follow the parent reference:

```javascript
// Find the parent of "Laptops" (id: 3)
const laptops = db.categories.findOne({ _id: 3 });
const parent = db.categories.findOne({ _id: laptops.parent });
console.log(parent.name); // "Computers"
```

## Traversing the Full Ancestor Path

Use a loop to walk up the tree from any node to the root:

```javascript
async function getAncestors(db, nodeId) {
  const path = [];
  let current = await db.categories.findOne({ _id: nodeId });

  while (current && current.parent !== null) {
    current = await db.categories.findOne({ _id: current.parent });
    path.unshift(current);
  }

  return path;
}

// Returns path: Electronics -> Computers -> Laptops for nodeId: 4
```

## Moving a Subtree

One major advantage of this pattern: moving an entire subtree only requires updating a single field on the subtree root.

```javascript
// Move "Computers" subtree under a new parent "Gadgets" (id: 7)
db.categories.updateOne(
  { _id: 2 },
  { $set: { parent: 7 } }
);
// All descendants automatically follow since they reference their direct parent
```

## Indexing for Performance

Create an index on the parent field to speed up child lookups:

```javascript
db.categories.createIndex({ parent: 1 });
```

For applications that also query by name frequently, add a compound index:

```javascript
db.categories.createIndex({ parent: 1, name: 1 });
```

## Fetching All Descendants with $graphLookup

MongoDB's `$graphLookup` aggregation stage makes it possible to retrieve an entire subtree in a single query:

```javascript
db.categories.aggregate([
  { $match: { _id: 1 } },
  {
    $graphLookup: {
      from: "categories",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "parent",
      as: "descendants",
      maxDepth: 10
    }
  }
]);
```

This returns the root node along with all descendants in the `descendants` array.

## Deleting a Node and Its Subtree

To safely delete a subtree, first collect all descendant IDs using `$graphLookup`, then delete them in bulk:

```javascript
const result = await db.categories.aggregate([
  { $match: { _id: 3 } },
  {
    $graphLookup: {
      from: "categories",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "parent",
      as: "subtree"
    }
  }
]).toArray();

const ids = [3, ...result[0].subtree.map(n => n._id)];
db.categories.deleteMany({ _id: { $in: ids } });
```

## Comparison with Other Tree Patterns

| Pattern | Best for | Query complexity |
|---|---|---|
| Parent References | Moving subtrees, upward traversal | Multiple queries for full subtree |
| Child References | Downward traversal | Multiple queries for ancestors |
| Materialized Paths | Full path display, prefix search | Single query |
| Nested Sets | Fast subtree reads | Expensive writes |

## Summary

The Parent References pattern is the simplest tree structure for MongoDB. Each node stores only its direct parent, making inserts and moves inexpensive. Use `$graphLookup` when you need full subtree queries in a single aggregation pipeline. This pattern works best when upward traversal and node relocation are the primary operations.
