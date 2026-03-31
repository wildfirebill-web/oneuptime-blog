# How to Export and Import Data with MongoDB Compass

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Compass, Export, Import, Data Migration

Description: Learn how to export filtered query results and import JSON or CSV data files into MongoDB collections using Compass.

---

## When to Use Compass for Data Transfer

Compass export and import are ideal for:
- Transferring small to medium datasets between environments
- Exporting query results for reporting or sharing
- Loading seed data during development
- Migrating data between collections

For large-scale production migrations, use `mongodump`/`mongorestore` or a dedicated ETL tool instead.

## Exporting Documents

### Export from the Documents Tab

1. Open a collection in Compass
2. (Optional) Apply a filter to export only matching documents
3. Click Collection in the menu bar, then Export Collection

The export dialog shows how many documents match your current filter.

### Export Formats

Compass supports two export formats:

**JSON** - preserves all MongoDB types:

```json
[
  {
    "_id": { "$oid": "65abc123..." },
    "name": "Widget A",
    "price": 9.99,
    "createdAt": { "$date": "2026-01-15T12:00:00.000Z" }
  }
]
```

**CSV** - flat format for spreadsheet tools. Nested objects and arrays are not well represented - best for flat collections only.

### Selecting Fields for Export

In the Export dialog, toggle which fields to include. This is useful for exporting a subset of fields without writing a projection query manually.

### Exporting Query Results

Apply a filter before opening the export dialog to export only matching documents:

```json
{ "status": "completed", "createdAt": { "$gte": { "$date": "2026-01-01T00:00:00Z" } } }
```

Then export - only the filtered documents are written to the file.

## Importing Documents

### Import from File

1. Click Collection in the menu bar, then Import Data
2. Select your file (JSON or CSV)
3. Configure options and click Import

### JSON Import Options

Compass accepts:
- A JSON array of documents
- Newline-delimited JSON (one document per line)

```json
{ "name": "Product A", "sku": "PA001", "stock": 100 }
{ "name": "Product B", "sku": "PB001", "stock": 50 }
```

### CSV Import Options

For CSV files, Compass shows a field mapping screen. You can:
- Set the data type for each column (String, Number, Date, Boolean, ObjectId)
- Ignore specific columns
- Specify a delimiter (comma or tab)

Example CSV:

```text
name,price,category,inStock
Widget A,9.99,tools,true
Widget B,14.99,tools,true
Gadget X,49.99,electronics,false
```

### Handling Import Errors

Compass shows a summary after import: how many documents were inserted and how many failed. Failed documents are typically caused by:
- Duplicate `_id` values
- Type mismatches in strongly-typed fields
- Documents exceeding the 16MB BSON size limit

You can stop on first error or continue importing valid documents and review failures afterward.

## Exporting the Full Collection Schema

Use the Schema tab to understand the collection before exporting, and copy the inferred schema as JSON for documentation:

```json
{
  "name": { "type": "string" },
  "price": { "type": "number" },
  "tags": { "type": "array" }
}
```

## Combining Export and Import for Environment Seeding

A typical workflow for seeding a development database:

```bash
# Export from staging (done in Compass)
# File saved as: staging-products.json

# Then import via Compass into your local dev MongoDB
# Or use mongoimport for scripting:
mongoimport \
  --uri "mongodb://localhost:27017/myapp" \
  --collection products \
  --file staging-products.json \
  --jsonArray
```

## Summary

Compass export and import cover the most common data transfer needs for development and small-scale migrations. Export with filters to get only the data you need, choose JSON to preserve MongoDB types, and use CSV for flat data destined for spreadsheets. For large datasets or automated pipelines, combine Compass for exploration with `mongoimport`/`mongoexport` for scripting.
