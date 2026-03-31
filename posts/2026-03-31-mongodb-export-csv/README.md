# How to Export MongoDB Data to CSV

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, CSV, Export, Mongoexport, Data Pipeline

Description: Learn how to export MongoDB collection data to CSV using mongoexport, the Node.js driver, and Python PyMongo for reports and data pipelines.

---

## Overview

Exporting MongoDB data to CSV enables sharing with spreadsheet tools, loading into data warehouses, and feeding external pipelines. MongoDB provides `mongoexport` for command-line exports and the native drivers for programmatic, filtered exports.

```mermaid
flowchart LR
    Mongo[(MongoDB Collection)] --> ME[mongoexport]
    Mongo --> Node[Node.js driver + csv-stringify]
    Mongo --> Py[PyMongo + pandas]
    ME & Node & Py --> CSV[CSV file / stream]
    CSV --> Excel[Excel / Google Sheets]
    CSV --> DW[Data Warehouse]
    CSV --> Report[Report]
```

## Method 1: mongoexport (Simplest)

`mongoexport` is part of MongoDB Database Tools.

```bash
# Export entire collection to CSV with specific fields
mongoexport \
  --uri "mongodb://localhost:27017/mydb" \
  --collection orders \
  --type csv \
  --fields "_id,customerId,total,status,createdAt" \
  --out orders.csv

# Export with a query filter (JSON format)
mongoexport \
  --uri "mongodb://localhost:27017/mydb" \
  --collection orders \
  --type csv \
  --fields "_id,customerId,total,status,createdAt" \
  --query '{"status":"shipped","createdAt":{"$gte":{"$date":"2026-01-01T00:00:00Z"}}}' \
  --out shipped_orders_2026.csv

# Export from Atlas
mongoexport \
  --uri "mongodb+srv://user:pass@cluster0.atlas.mongodb.net/mydb" \
  --collection products \
  --type csv \
  --fields "sku,name,price,stock,category" \
  --out products.csv \
  --ssl

# Limit number of documents exported
mongoexport \
  --uri "mongodb://localhost:27017/mydb" \
  --collection events \
  --type csv \
  --fields "ts,userId,eventType,pageUrl" \
  --limit 10000 \
  --sort '{"ts":-1}' \
  --out recent_events.csv
```

## Method 2: Node.js with csv-stringify

For programmatic exports with custom logic, aggregation, and streaming large datasets to disk.

```bash
npm install csv-stringify mongodb
```

```javascript
const { MongoClient } = require("mongodb");
const { stringify }   = require("csv-stringify");
const fs              = require("fs");

const client = new MongoClient(process.env.MONGO_URI);

async function exportToCSV(dbName, collName, filter, outputPath) {
  await client.connect();
  const collection = client.db(dbName).collection(collName);

  const writeStream  = fs.createWriteStream(outputPath);
  const csvStringify = stringify({ header: true });

  csvStringify.pipe(writeStream);

  const cursor = collection.find(filter, {
    projection: { _id: 1, customerId: 1, total: 1, status: 1, createdAt: 1 }
  }).sort({ createdAt: -1 });

  let count = 0;
  for await (const doc of cursor) {
    csvStringify.write({
      id:         doc._id.toString(),
      customerId: doc.customerId?.toString() || "",
      total:      doc.total,
      status:     doc.status,
      createdAt:  doc.createdAt?.toISOString() || ""
    });
    count++;
    if (count % 1000 === 0) process.stdout.write(`\rExported: ${count}`);
  }

  csvStringify.end();

  await new Promise((resolve, reject) => {
    writeStream.on("finish", resolve);
    writeStream.on("error", reject);
  });

  console.log(`\nExported ${count} documents to ${outputPath}`);
  await client.close();
}

exportToCSV(
  "mydb",
  "orders",
  { status: "shipped", createdAt: { $gte: new Date("2026-01-01") } },
  "./shipped_orders.csv"
);
```

## Method 3: Export Aggregation Pipeline Results

When the exported data requires joining, grouping, or transforming, run an aggregation pipeline and stream results to CSV.

```javascript
async function exportAggregationToCSV(db, pipeline, outputPath) {
  const { stringify } = require("csv-stringify");
  const fs = require("fs");

  const writeStream  = fs.createWriteStream(outputPath);
  const csvStringify = stringify({ header: true });
  csvStringify.pipe(writeStream);

  const cursor = db.collection("orders").aggregate(pipeline, {
    allowDiskUse: true  // allow large aggregations to spill to disk
  });

  let count = 0;
  for await (const doc of cursor) {
    csvStringify.write(doc);
    count++;
  }

  csvStringify.end();

  await new Promise((res, rej) => {
    writeStream.on("finish", res);
    writeStream.on("error", rej);
  });

  console.log(`Exported ${count} rows to ${outputPath}`);
}

// Example: monthly revenue export
const pipeline = [
  {
    $group: {
      _id: {
        year:  { $year: "$createdAt" },
        month: { $month: "$createdAt" }
      },
      revenue:    { $sum: "$total" },
      orderCount: { $sum: 1 },
      avgOrder:   { $avg: "$total" }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } },
  {
    $project: {
      _id: 0,
      year:        "$_id.year",
      month:       "$_id.month",
      revenue:     { $round: ["$revenue", 2] },
      order_count: "$orderCount",
      avg_order:   { $round: ["$avgOrder", 2] }
    }
  }
];

await exportAggregationToCSV(db, pipeline, "./monthly_revenue.csv");
```

## Method 4: Python with PyMongo and pandas

```python
import pandas as pd
from pymongo import MongoClient
from datetime import datetime
import os

client = MongoClient(os.environ["MONGO_URI"])
db = client["mydb"]

# Export collection with a filter
cursor = db["orders"].find(
    {"status": "shipped", "createdAt": {"$gte": datetime(2026, 1, 1)}},
    {"_id": 1, "customerId": 1, "total": 1, "status": 1, "createdAt": 1}
).sort("createdAt", -1)

df = pd.DataFrame(list(cursor))

# Convert ObjectId fields to strings
for col in ["_id", "customerId"]:
    if col in df.columns:
        df[col] = df[col].astype(str)

df.to_csv("shipped_orders.csv", index=False)
print(f"Exported {len(df)} rows")
client.close()
```

## Method 5: Export to CSV from mongosh

```javascript
// mongosh script to export to CSV (prints to stdout, redirect to file)
// mongosh mydb --eval "$(cat export.js)" > output.csv

const cursor = db.orders.find(
  { status: "shipped" },
  { _id: 1, total: 1, status: 1, createdAt: 1 }
).sort({ createdAt: -1 }).limit(1000);

// Print header
print("_id,total,status,createdAt");

// Print rows
cursor.forEach((doc) => {
  print([
    doc._id.toString(),
    doc.total,
    doc.status,
    doc.createdAt.toISOString()
  ].join(","));
});
```

```bash
mongosh mydb export.js > output.csv
```

## Handling Special Characters in CSV

```javascript
// csv-stringify handles quoting and escaping automatically
// For fields that may contain commas, quotes, or newlines:
csvStringify.write({
  description: doc.description,  // will be quoted if it contains commas
  notes: doc.notes                // newlines inside strings are handled correctly
});
```

## Summary

Export MongoDB data to CSV using `mongoexport --type csv --fields` for simple collection dumps, or use the Node.js driver with `csv-stringify` for streaming large datasets and applying transformations. Run aggregation pipelines before exporting to produce pre-grouped or joined output. In Python, use PyMongo with pandas for DataFrame-based transformation before writing CSV. Always stream large exports rather than loading the entire collection into memory.
