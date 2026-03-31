# How to Use DataGrip for MongoDB Development

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, DataGrip, Developer Tool, Query, Operation

Description: Learn how to use JetBrains DataGrip to connect to MongoDB, browse schemas, run aggregation pipelines, export data, and manage indexes with a professional IDE experience.

---

## Why DataGrip for MongoDB?

DataGrip is JetBrains' dedicated database IDE. It supports MongoDB alongside relational databases, giving you a unified tool for polyglot database teams. Key features include intelligent query completion, result set editing, data export, explain plan visualization, and version-controlled query history.

## Setting Up a MongoDB Connection

1. Open DataGrip and go to Database > New > Data Source > MongoDB
2. Enter connection details:

```text
Host:     localhost
Port:     27017
Database: shop
```

For Atlas, use the full SRV connection string:

```text
URL: mongodb+srv://admin:password@mycluster.mongodb.net/shop?authSource=admin
```

3. Click "Test Connection" and download the driver if prompted
4. Click "Apply" then "OK"

DataGrip saves the connection and introspects the schema in the background.

## Browsing Collections and Schemas

The Database Explorer panel shows:

```text
MongoDB - localhost
  shop (database)
    orders (collection, ~50,000 docs)
    products (collection, ~2,000 docs)
    reviews (collection, ~120,000 docs)
```

Click a collection to open it in the Data Editor. DataGrip renders documents in a tabular view with expandable nested objects and arrays.

## Writing and Running Queries

Open a query console (Ctrl+Shift+Q or right-click the database > New > Query Console):

```javascript
// Find all pending orders from the last 7 days
db.orders.find({
  status: "pending",
  createdAt: { $gte: new Date(Date.now() - 7 * 86400000) }
}).sort({ createdAt: -1 }).limit(50)
```

DataGrip provides:
- Autocompletion for collection names and field names
- Syntax highlighting for MongoDB shell syntax
- Error highlighting for invalid operators

## Running Aggregation Pipelines

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  { $group: {
    _id: "$customerId",
    orderCount: { $sum: 1 },
    totalSpend: { $sum: "$total" }
  }},
  { $sort: { totalSpend: -1 } },
  { $limit: 10 }
])
```

Results display in the Output panel with pagination for large result sets.

## Editing Documents In-Place

In the Data Editor, double-click a cell to edit a field value. Use the toolbar to:
- Add new documents
- Delete selected documents
- Submit or rollback changes

DataGrip generates the appropriate `updateOne` or `deleteOne` operation based on your edits.

## Exporting Data

Right-click a collection or query result and select Export Data:

```text
Format options: JSON, CSV, TSV, SQL INSERT (for relational export)
Scope:          All rows, current page, or selected rows
Filter:         Apply a find query before export
```

Example CLI equivalent of a DataGrip export:

```bash
mongoexport \
  --uri "mongodb://localhost:27017/shop" \
  --collection orders \
  --query '{"status":"paid"}' \
  --out paid-orders.json
```

## Explain Plans in DataGrip

Add `.explain("executionStats")` to any `find` or `aggregate` query and DataGrip renders the plan:

```javascript
db.orders.find({ status: "paid" }).explain("executionStats")
```

The output shows winning plan stages, `docsExamined`, `keysExamined`, and execution time.

## Summary

DataGrip provides a professional MongoDB development environment with schema browsing, intelligent query completion, aggregation pipeline support, in-place document editing, flexible data export, and explain plan visualization. It is particularly valuable for teams working across multiple database systems, as it unifies MySQL, PostgreSQL, Redis, and MongoDB in one tool.
