# How to Use mongoimport to Import CSV Data into MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, mongoimport, CSV, Import, Data Transfer

Description: Import CSV files into MongoDB using mongoimport with field mapping, type coercion, and upsert options for flexible data ingestion.

---

## When to Use mongoimport for CSV

`mongoimport` is the standard tool for importing CSV data into MongoDB. Common use cases include:

- Migrating data from SQL databases exported as CSV
- Loading data from spreadsheets or reports
- Bulk importing product catalogs, user lists, or event logs
- Seeding databases for development or testing

## Basic CSV Import

```bash
mongoimport \
  --host localhost:27017 \
  --username admin \
  --password secret \
  --authenticationDatabase admin \
  --db myapp \
  --collection users \
  --type csv \
  --headerline \
  --file /imports/users.csv
```

`--headerline` tells `mongoimport` to use the first row as field names.

## Import CSV with Explicit Field Names

If the CSV file has no header row, specify field names manually:

```bash
mongoimport \
  --db myapp \
  --collection products \
  --type csv \
  --fields "sku,name,price,category,stock" \
  --file /imports/products.csv
```

## Handling Duplicate Records

### Drop and Re-import

```bash
mongoimport \
  --db myapp \
  --collection products \
  --type csv \
  --headerline \
  --drop \
  --file /imports/products.csv
```

### Upsert Mode

Update existing documents and insert new ones based on a unique field:

```bash
mongoimport \
  --db myapp \
  --collection products \
  --type csv \
  --headerline \
  --upsert \
  --upsertFields sku \
  --file /imports/products.csv
```

## Type Coercion

By default, all CSV fields are imported as strings. Use `--columnsHaveTypes` to specify types:

```bash
mongoimport \
  --db myapp \
  --collection orders \
  --type csv \
  --fields "orderId.string(),amount.double(),quantity.int32(),createdAt.date(2006-01-02)" \
  --columnsHaveTypes \
  --file /imports/orders.csv
```

Supported types: `string()`, `int32()`, `int64()`, `double()`, `boolean()`, `date()`, `date_go()`, `date_ms()`, `date_oracle()`, `objectId()`, `decimal()`.

## Importing Large CSV Files

For large files, `mongoimport` imports in batches. Increase batch size for better performance:

```bash
mongoimport \
  --db myapp \
  --collection logs \
  --type csv \
  --headerline \
  --batchSize 1000 \
  --numInsertionWorkers 4 \
  --file /imports/large-logs.csv
```

## Ignoring Blank Fields

Skip empty CSV fields to avoid inserting null values:

```bash
mongoimport \
  --db myapp \
  --collection users \
  --type csv \
  --headerline \
  --ignoreBlanks \
  --file /imports/users.csv
```

## Sample CSV Format

```text
name,email,age,signupDate
Alice Smith,alice@example.com,28,2026-01-15
Bob Jones,bob@example.com,34,2026-01-16
Carol White,carol@example.com,,2026-01-17
```

With `--ignoreBlanks`, Carol's missing age field will not be inserted as null.

## Verifying the Import

After import, confirm the document count:

```javascript
use myapp
db.users.countDocuments()
db.users.findOne()
```

Check for type correctness if using `--columnsHaveTypes`:

```javascript
db.orders.findOne({ amount: { $gt: 100 } })
```

## Summary

`mongoimport` with `--type csv` and `--headerline` is the simplest way to import CSV data. Use `--upsert` and `--upsertFields` for incremental updates, `--columnsHaveTypes` for proper data types, and `--ignoreBlanks` to skip empty fields. For large files, increase `--batchSize` and `--numInsertionWorkers` for faster throughput.
