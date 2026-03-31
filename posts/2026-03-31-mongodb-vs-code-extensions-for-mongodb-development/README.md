# How to Use VS Code Extensions for MongoDB Development

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, VS Code, Developer Tool, Productivity, Extension

Description: Learn how to use the MongoDB for VS Code extension to connect, query, and manage MongoDB databases directly from your editor with IntelliSense and Playgrounds.

---

## The MongoDB for VS Code Extension

The official MongoDB for VS Code extension (published by MongoDB Inc.) integrates the full MongoDB development workflow into your editor. It provides a connection manager, document explorer, query execution with IntelliSense, and MongoDB Playgrounds for experimenting with aggregation pipelines.

Install it from the VS Code Extensions Marketplace:

```bash
code --install-extension mongodb.mongodb-vscode
```

Or search for "MongoDB for VS Code" in the Extensions panel (Ctrl+Shift+X).

## Connecting to a MongoDB Instance

1. Open the MongoDB panel from the activity bar (the leaf icon)
2. Click "Add Connection"
3. Enter your connection string:

```text
mongodb://localhost:27017
mongodb+srv://username:password@cluster.mongodb.net/mydb
```

For Atlas connections with SRV, the extension automatically handles DNS resolution. You can store multiple connections and switch between them.

## Exploring Collections

Once connected, expand the tree to browse:

```text
MyCluster
  mydb
    collections
      orders (1,234 docs)
      products (567 docs)
    indexes
      orders._id_
      orders.status_1_createdAt_-1
```

Click any collection to open a document viewer. Use the filter bar to query:

```javascript
{ status: "pending", createdAt: { $gte: ISODate("2026-03-01") } }
```

## MongoDB Playgrounds

Playgrounds are JavaScript files that run against your connected cluster. Create one with Ctrl+Shift+P > "MongoDB: Create New Playground":

```javascript
// playground-1.mongodb.js
use("shop");

// Find top 5 products by revenue
db.getCollection("orders").aggregate([
  { $match: { status: "paid" } },
  { $unwind: "$items" },
  { $group: {
    _id: "$items.productId",
    revenue: { $sum: { $multiply: ["$items.price", "$items.qty"] } }
  }},
  { $sort: { revenue: -1 } },
  { $limit: 5 }
]);
```

Run with Ctrl+Shift+P > "MongoDB: Run All Playground Blocks" or click the run button. Results appear in the output panel.

## IntelliSense in Playgrounds

The extension provides autocompletion for:
- Collection names after `db.getCollection("`
- MongoDB operators like `$match`, `$group`, `$lookup`
- Field names inferred from your collection's schema

## Schema Inspection

Right-click a collection and select "View as table" to see field types inferred from a sample of documents. This is useful for understanding the shape of legacy collections.

## Export and Import Documents

Right-click a collection to export documents:
- Export as JSON or CSV
- Apply a filter to export a subset
- Import documents from a JSON file

```javascript
// Export from Playground
use("shop");
const orders = db.orders.find({ status: "cancelled" }).toArray();
// Copy JSON output for import elsewhere
```

## Running Explain Plans

In a playground, use `.explain()` to see the query plan:

```javascript
use("shop");
db.orders.explain("executionStats").find({ status: "pending" });
```

The extension renders the explain plan output in the results panel.

## Summary

The MongoDB for VS Code extension brings connection management, document browsing, Playground-based query execution, and IntelliSense for MongoDB operators directly into your editor. Use Playgrounds to prototype and test aggregation pipelines before embedding them in application code, the document viewer for quick data inspection, and the schema view to understand field distributions in unfamiliar collections.
