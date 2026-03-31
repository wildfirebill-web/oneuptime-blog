# How to Implement the Tree Pattern in MongoDB (Child References)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Schema Design, Tree Pattern, Child References, Hierarchical Data

Description: Learn how to implement the child references tree pattern in MongoDB to model hierarchical data like category trees, org charts, and file systems.

---

## Overview

The child references tree pattern models hierarchical data by storing each node with an array of direct child node identifiers. Each document knows its immediate children but not its ancestors. This pattern is efficient for top-down traversal (finding all children of a node) but requires multiple queries for bottom-up operations.

## Child References Pattern Structure

```javascript
// Each document contains an array of its children's IDs
{
  _id: "node_id",
  name: "Node Name",
  children: ["child1_id", "child2_id", "child3_id"]
}
```

## Setting Up a Category Tree

```javascript
db.categories.insertMany([
  { _id: "root", name: "All Products", children: ["electronics", "clothing", "home"] },
  { _id: "electronics", name: "Electronics", children: ["laptops", "phones", "tablets"] },
  { _id: "clothing", name: "Clothing", children: ["mens", "womens"] },
  { _id: "home", name: "Home & Garden", children: ["furniture", "kitchen"] },
  { _id: "laptops", name: "Laptops", children: [] },
  { _id: "phones", name: "Phones", children: ["smartphones", "accessories"] },
  { _id: "tablets", name: "Tablets", children: [] },
  { _id: "mens", name: "Men's Clothing", children: [] },
  { _id: "womens", name: "Women's Clothing", children: [] },
  { _id: "smartphones", name: "Smartphones", children: [] },
  { _id: "accessories", name: "Phone Accessories", children: [] },
  { _id: "furniture", name: "Furniture", children: [] },
  { _id: "kitchen", name: "Kitchen", children: [] }
])

// Index for efficient parent lookup
db.categories.createIndex({ children: 1 })
```

## Finding Direct Children

```javascript
// Get all direct children of "electronics"
const electronics = db.categories.findOne({ _id: "electronics" });
db.categories.find({ _id: { $in: electronics.children } })
```

## Finding the Parent of a Node

```javascript
// Find which category contains "laptops" as a child
db.categories.findOne({ children: "laptops" })
// Returns: { _id: "electronics", name: "Electronics", children: [...] }
```

## Recursive Traversal with Application Code

Child references require multiple queries for deep traversal. Here is a recursive fetch:

```javascript
async function getAllDescendants(db, nodeId) {
  const descendants = [];
  const queue = [nodeId];

  while (queue.length > 0) {
    const currentId = queue.shift();
    const node = await db.collection("categories").findOne({ _id: currentId });

    if (node && node.children && node.children.length > 0) {
      descendants.push(...node.children);
      queue.push(...node.children);
    }
  }

  return descendants;
}

// Get all descendants of "electronics"
const allUnderElectronics = await getAllDescendants(db, "electronics");
```

## Using $graphLookup for Recursive Traversal

MongoDB's `$graphLookup` can traverse child references automatically:

```javascript
db.categories.aggregate([
  { $match: { _id: "electronics" } },
  {
    $graphLookup: {
      from: "categories",
      startWith: "$children",
      connectFromField: "children",
      connectToField: "_id",
      as: "allDescendants",
      maxDepth: 10,
      depthField: "depth"
    }
  },
  {
    $project: {
      name: 1,
      descendantNames: "$allDescendants.name"
    }
  }
])
```

## Finding All Ancestors ($graphLookup for Parent Reference)

To find ancestors, you need to traverse via the parent relationship. If you also store `parent` field:

```javascript
// Add parent reference for bidirectional traversal
db.categories.insertMany([
  { _id: "laptops", name: "Laptops", parent: "electronics", children: [] },
  { _id: "electronics", name: "Electronics", parent: "root", children: ["laptops", "phones"] }
])

// Find all ancestors of "laptops"
db.categories.aggregate([
  { $match: { _id: "laptops" } },
  {
    $graphLookup: {
      from: "categories",
      startWith: "$parent",
      connectFromField: "parent",
      connectToField: "_id",
      as: "ancestors"
    }
  },
  { $project: { name: 1, breadcrumb: "$ancestors.name" } }
])
```

## Adding a Node to the Tree

```javascript
// Add a new subcategory "gaming-laptops" under "laptops"
db.categories.insertOne({
  _id: "gaming-laptops",
  name: "Gaming Laptops",
  children: []
})

db.categories.updateOne(
  { _id: "laptops" },
  { $push: { children: "gaming-laptops" } }
)
```

## Moving a Node

```javascript
// Move "tablets" from "electronics" to a new "mobile" category
// Step 1: Remove from old parent
db.categories.updateOne(
  { _id: "electronics" },
  { $pull: { children: "tablets" } }
)

// Step 2: Add to new parent
db.categories.updateOne(
  { _id: "mobile" },
  { $push: { children: "tablets" } }
)
```

## Tree Pattern Comparison

```text
| Pattern            | Find children | Find parent | Find ancestors | Find subtree |
|--------------------|---------------|-------------|----------------|--------------|
| Child references   | Easy          | One query   | Requires walk  | $graphLookup |
| Parent reference   | One query     | Easy        | $graphLookup   | One query    |
| Array of ancestors | Easy          | Easy        | Easy           | One query    |
| Nested sets        | Complex       | Complex     | One query      | One query    |
```

## Summary

The child references tree pattern stores each node with an array of its direct children's IDs. It enables efficient top-down traversal (finding children of a node) and is easy to update when adding new leaf nodes. Use `$graphLookup` to traverse the tree recursively without multiple round trips. For collections that need efficient bottom-up queries (finding ancestors) in addition to top-down, consider combining child references with a parent field, or use the array of ancestors pattern instead.
