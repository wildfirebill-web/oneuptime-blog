# How to Export a Collection to CSV in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Export, CSV, Mongoexport, Database

Description: Learn how to export a MongoDB collection to CSV using mongoexport, with field selection, query filtering, and tips for handling nested documents.

---

## Overview

Exporting MongoDB data to CSV is useful for sharing with stakeholders, loading into spreadsheet tools, or feeding into analytics pipelines. The `mongoexport` tool handles CSV export with flexible field and query options.

## Basic CSV Export

CSV exports require you to specify the fields to include with `--fields`:

```bash
mongoexport \
  --uri "mongodb://localhost:27017/mydb" \
  --collection customers \
  --type csv \
  --fields "name,email,city,country" \
  --out customers.csv
```

The output file will have a header row followed by one row per document.

## Filtering Documents

Export only a subset of documents using `--query`:

```bash
mongoexport \
  --uri "mongodb://localhost:27017/mydb" \
  --collection orders \
  --type csv \
  --fields "orderId,customerId,total,status,createdAt" \
  --query '{ "status": "shipped" }' \
  --out shipped_orders.csv
```

## Exporting Nested Fields

Use dot notation to access nested document fields:

```bash
mongoexport \
  --uri "mongodb://localhost:27017/mydb" \
  --collection users \
  --type csv \
  --fields "name,address.city,address.country,profile.age" \
  --out users_locations.csv
```

## Adding a Sort Order

Use `--sort` to control the row order in the output:

```bash
mongoexport \
  --uri "mongodb://localhost:27017/mydb" \
  --collection products \
  --type csv \
  --fields "sku,name,price,stock" \
  --sort '{ "price": 1 }' \
  --out products_sorted.csv
```

## Exporting with Authentication

```bash
mongoexport \
  --uri "mongodb://admin:password@localhost:27017/mydb?authSource=admin" \
  --collection invoices \
  --type csv \
  --fields "invoiceId,amount,dueDate,paid" \
  --out invoices.csv
```

## Programmatic CSV Export with Python

For transforming data during export, use PyMongo and the `csv` module:

```python
import csv
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017/")
db = client["mydb"]
collection = db["customers"]

with open("customers_export.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["name", "email", "city", "country"])
    for doc in collection.find({}, {"name": 1, "email": 1, "address": 1}):
        writer.writerow([
            doc.get("name", ""),
            doc.get("email", ""),
            doc.get("address", {}).get("city", ""),
            doc.get("address", {}).get("country", "")
        ])

print("CSV export complete")
```

## Handling Arrays in CSV

CSV format cannot represent arrays natively. Use the aggregation pipeline to flatten arrays before exporting:

```bash
mongoexport \
  --uri "mongodb://localhost:27017/mydb" \
  --collection orders \
  --type csv \
  --fields "orderId,tags" \
  --out orders_tags.csv
```

For multi-value fields, consider joining array elements into a delimited string using `$reduce` in an aggregation pipeline first, then exporting the result collection.

## Summary

Use `mongoexport` with `--type csv` and `--fields` to export specific fields from a MongoDB collection to CSV. Add `--query` to filter documents and `--sort` to control row ordering. For complex transformations - especially flattening nested documents or arrays - use the Python driver or an aggregation pipeline before exporting.
