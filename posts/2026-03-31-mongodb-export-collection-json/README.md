# How to Export a Collection to JSON in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Export, JSON, Mongoexport, Database

Description: Learn how to export a MongoDB collection to JSON using mongoexport, with options for filtering, field selection, and handling large collections efficiently.

---

## Overview

MongoDB's `mongoexport` tool writes collection data to a file in JSON or CSV format. It is ideal for ad-hoc exports, data migrations, and sharing subsets of data with external systems.

## Basic Export to JSON

```bash
mongoexport \
  --uri "mongodb://localhost:27017/mydb" \
  --collection products \
  --out products.json
```

This exports every document in the `products` collection to `products.json` in NDJSON format (one document per line).

## Exporting as a JSON Array

To produce a proper JSON array instead of NDJSON, add `--jsonArray`:

```bash
mongoexport \
  --uri "mongodb://localhost:27017/mydb" \
  --collection products \
  --jsonArray \
  --out products_array.json
```

The output file will be a valid JSON array, suitable for use with tools that expect a JSON array input.

## Filtering Documents During Export

Use `--query` to export only documents matching a condition:

```bash
mongoexport \
  --uri "mongodb://localhost:27017/mydb" \
  --collection orders \
  --query '{ "status": "completed", "total": { "$gt": 100 } }' \
  --out completed_orders.json
```

## Selecting Specific Fields

Use `--fields` to include only certain fields in the export:

```bash
mongoexport \
  --uri "mongodb://localhost:27017/mydb" \
  --collection users \
  --fields "name,email,createdAt" \
  --out users_emails.json
```

## Exporting with Authentication

```bash
mongoexport \
  --uri "mongodb://admin:password@localhost:27017/mydb?authSource=admin" \
  --collection inventory \
  --out inventory.json
```

## Exporting to a Remote Atlas Cluster

```bash
mongoexport \
  --uri "mongodb+srv://user:pass@cluster0.example.mongodb.net/mydb" \
  --collection products \
  --out products_atlas.json
```

## Programmatic Export with Node.js

For more control, use the driver to stream documents directly to a file:

```javascript
const { MongoClient } = require("mongodb");
const fs = require("fs");

async function exportCollection() {
  const client = new MongoClient("mongodb://localhost:27017");
  await client.connect();
  const collection = client.db("mydb").collection("products");
  const stream = fs.createWriteStream("products_export.json");

  stream.write("[\n");
  let first = true;
  const cursor = collection.find({});
  for await (const doc of cursor) {
    if (!first) stream.write(",\n");
    stream.write(JSON.stringify(doc));
    first = false;
  }
  stream.write("\n]");
  stream.end();
  await client.close();
  console.log("Export complete");
}

exportCollection();
```

## Summary

Use `mongoexport` for quick JSON exports with optional query filters and field projections. Add `--jsonArray` when the target system expects a JSON array. For large collections, NDJSON (the default) is more memory-efficient. Use the MongoDB driver for programmatic exports that require custom transformation or streaming to external destinations.
