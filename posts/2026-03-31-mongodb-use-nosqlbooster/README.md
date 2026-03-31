# How to Use NoSQLBooster for MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, NoSQLBooster, Developer Tool, GUI, Query

Description: Learn how to use NoSQLBooster for MongoDB to write queries with IntelliSense, run fluent API queries, visualize results, and manage collections from a feature-rich GUI.

---

## What Is NoSQLBooster?

NoSQLBooster (formerly MongoBooster) is a cross-platform GUI tool for MongoDB with one of the most complete IntelliSense implementations among MongoDB clients. It supports the MongoDB shell API, the fluent API, ES2020+ JavaScript, and embedded Node.js for scripting tasks that go beyond simple queries.

## Connecting to MongoDB

1. Launch NoSQLBooster and click "Connect"
2. Enter the connection string:

```text
mongodb://localhost:27017
```

For Atlas:

```text
mongodb+srv://username:password@cluster.mongodb.net/
```

3. Select your authentication method (SCRAM-SHA-256 for Atlas)
4. Click "Test" then "Save & Connect"

## IntelliSense in the Query Editor

NoSQLBooster's IntelliSense is context-aware. As you type, it suggests:
- Database names after `use()`
- Collection names after `db.`
- Method names like `.find()`, `.aggregate()`, `.insertOne()`
- Field names inferred from your collection's schema
- MongoDB operator names after `{`

```javascript
use("shop");
db.orders
  .find({ status: "paid", total: { $gte: 100 } })
  .sort({ createdAt: -1 })
  .limit(20)
  .project({ customerId: 1, total: 1, createdAt: 1 });
```

## Fluent API Queries

NoSQLBooster supports a jQuery-like fluent API on top of the standard MongoDB shell:

```javascript
use("shop");
// Fluent API with method chaining
db.orders
  .where("status").eq("paid")
  .where("total").gte(100)
  .orderBy("createdAt", "desc")
  .limit(20)
  .select("customerId", "total", "createdAt");
```

This is useful for developers coming from SQL ORMs who find the fluent style more readable.

## Running Aggregation Pipelines

```javascript
use("shop");
db.orders.aggregate([
  { $match: { status: "paid" } },
  { $group: {
    _id: { $dateToString: { format: "%Y-%m-%d", date: "$createdAt" } },
    revenue: { $sum: "$total" },
    orders: { $sum: 1 }
  }},
  { $sort: { _id: -1 } },
  { $limit: 30 }
]);
```

## Result Visualization

NoSQLBooster can render query results as:
- JSON tree
- Table view
- Chart (bar, line, pie) - select a numeric field and click Chart

Example: visualize daily revenue by selecting `revenue` as the Y-axis and `_id` (date) as the X-axis.

## Scripting with Embedded Node.js

NoSQLBooster includes a full Node.js runtime. Write multi-step scripts:

```javascript
use("shop");

// Migrate data with transformation
const cursor = db.orders.find({ migratedV2: { $ne: true } }).limit(500);
let updated = 0;

while (cursor.hasNext()) {
  const order = cursor.next();
  const newFormat = {
    ...order,
    customerRef: { id: order.customerId, name: order.customerName },
    migratedV2: true
  };
  db.ordersV2.insertOne(newFormat);
  db.orders.updateOne({ _id: order._id }, { $set: { migratedV2: true } });
  updated++;
}

print(`Migrated ${updated} orders`);
```

## Export and Import

Right-click a collection to export:

```text
Format: JSON, CSV, TSV, BSON
Filter: Apply a query before export
Encoding: UTF-8
```

## Keyboard Shortcuts

```text
Ctrl+Enter       - Run current query
Ctrl+Shift+Enter - Run all queries
Ctrl+Space       - Trigger IntelliSense
Ctrl+/           - Toggle comment
F9               - Format document
```

## Summary

NoSQLBooster stands out for its comprehensive IntelliSense that covers collection names, field names, and operator names, its fluent query API for developers who prefer chained method syntax, built-in result chart visualization, and embedded Node.js for complex scripting scenarios. It is a strong choice for developers who spend significant time writing MongoDB queries and want IDE-quality assistance in a dedicated database tool.
