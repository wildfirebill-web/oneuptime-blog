# How to Use mongoexport to Export Data to CSV in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, mongoexport, CSV, Export, Data Transfer

Description: Use mongoexport to export MongoDB collection data to CSV format for use in spreadsheets, data pipelines, and reporting tools.

---

## When to Use mongoexport for CSV

`mongoexport` with CSV output is useful when you need to:
- Share data with non-technical stakeholders using spreadsheets
- Import MongoDB data into SQL databases or ETL pipelines
- Generate reports from collection data
- Feed data into analytics tools like Tableau or Power BI

Note: `mongoexport` is not intended for full backups - use `mongodump` for that.

## Basic CSV Export

Export a collection to CSV with specific fields:

```bash
mongoexport \
  --host localhost:27017 \
  --username admin \
  --password secret \
  --authenticationDatabase admin \
  --db myapp \
  --collection users \
  --type csv \
  --fields name,email,createdAt \
  --out /exports/users.csv
```

The `--fields` parameter is required for CSV output - you must specify which fields to include.

## Export with a Header Row

By default, `mongoexport` includes a header row with field names:

```bash
mongoexport \
  --db myapp \
  --collection orders \
  --type csv \
  --fields orderId,customerId,amount,status,createdAt \
  --out /exports/orders.csv
```

Output preview:

```text
orderId,customerId,amount,status,createdAt
ORD-001,user123,49.99,shipped,2026-01-15T10:30:00Z
ORD-002,user456,129.00,pending,2026-01-16T14:22:00Z
```

## Export with Query Filter

Export only matching documents using `--query`:

```bash
mongoexport \
  --db myapp \
  --collection orders \
  --type csv \
  --fields orderId,customerId,amount,status \
  --query '{"status": "completed", "amount": {"$gt": 100}}' \
  --out /exports/large-orders.csv
```

## Export Nested Fields

For nested documents, use dot notation in `--fields`:

```bash
mongoexport \
  --db myapp \
  --collection users \
  --type csv \
  --fields "name,email,address.city,address.country" \
  --out /exports/users-with-address.csv
```

## Export with Date Range

```bash
mongoexport \
  --db myapp \
  --collection events \
  --type csv \
  --fields eventId,type,userId,timestamp \
  --query '{"timestamp": {"$gte": {"$date": "2026-01-01T00:00:00Z"}, "$lt": {"$date": "2026-02-01T00:00:00Z"}}}' \
  --out /exports/january-events.csv
```

## Export with Sort and Limit

```bash
mongoexport \
  --db myapp \
  --collection products \
  --type csv \
  --fields sku,name,price,stock \
  --sort '{"price": -1}' \
  --limit 100 \
  --out /exports/top-100-products.csv
```

## Handling Special Characters

If your data contains commas or quotes, `mongoexport` properly escapes them per RFC 4180. For multi-line text fields, the values are wrapped in quotes.

## Compressing the Output

Pipe to gzip for large exports:

```bash
mongoexport \
  --db myapp \
  --collection orders \
  --type csv \
  --fields orderId,amount,status \
  | gzip > /exports/orders.csv.gz
```

## Automating CSV Exports

Schedule regular exports with cron:

```bash
# /etc/cron.daily/mongo-csv-export
#!/bin/bash
DATE=$(date +%Y%m%d)
mongoexport \
  --db myapp \
  --collection orders \
  --type csv \
  --fields orderId,customerId,amount,status,createdAt \
  --query "{\"createdAt\": {\"\$gte\": {\"\$date\": \"$(date -d yesterday +%Y-%m-%dT00:00:00Z)\"}}}" \
  --out "/exports/orders-${DATE}.csv"
```

## Summary

`mongoexport` with `--type csv` is the standard way to export MongoDB collections to CSV. Always specify `--fields` for CSV exports, use `--query` to filter data, and dot notation to flatten nested fields. For large datasets, pipe through gzip compression and schedule exports via cron for automated reporting workflows.
