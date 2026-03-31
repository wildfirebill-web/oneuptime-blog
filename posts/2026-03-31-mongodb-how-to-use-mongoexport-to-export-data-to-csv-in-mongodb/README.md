# How to Use mongoexport to Export Data to CSV in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongoexport, CSV, Data Export

Description: Learn how to export MongoDB collection data to CSV format using mongoexport, including field selection, filtering, handling nested documents, and Excel compatibility.

---

## Overview

`mongoexport` supports exporting MongoDB data to CSV format, making it easy to share data with business analysts, import into spreadsheet applications, or load into other databases. Unlike JSON export, CSV requires you to specify which fields to export since MongoDB's flexible schema means different documents can have different fields.

## Basic CSV Export

```bash
# Export collection to CSV with specified fields
mongoexport   --uri "mongodb://admin:secret@localhost:27017"   --db mydb   --collection orders   --type csv   --fields "orderId,customerId,status,total,createdAt"   --out /tmp/orders.csv
```

The `--type csv` flag enables CSV mode. The `--fields` flag is required for CSV export because MongoDB needs to know which fields to include as columns.

## CSV Output Format

The output includes a header row by default:

```text
orderId,customerId,status,total,createdAt
ORD-001,CUST-123,pending,99.99,2026-03-31T10:00:00.000Z
ORD-002,CUST-124,completed,49.99,2026-03-30T15:30:00.000Z
ORD-003,CUST-125,pending,149.99,2026-03-31T09:15:00.000Z
```

## Selecting Fields for Export

```bash
# Export with specific fields
mongoexport   --uri "mongodb://admin:secret@localhost:27017"   --db mydb   --collection orders   --type csv   --fields "orderId,status,total"   --out /tmp/orders-summary.csv

# Use a fields file for long field lists
cat > /tmp/fields.txt << 'EOF'
orderId
customerId
status
total
shippingAddress.city
shippingAddress.country
createdAt
updatedAt
EOF

mongoexport   --uri "mongodb://admin:secret@localhost:27017"   --db mydb   --collection orders   --type csv   --fieldFile /tmp/fields.txt   --out /tmp/orders-detailed.csv
```

## Accessing Nested Fields

Access nested document fields using dot notation:

```bash
# Export nested fields
mongoexport   --uri "mongodb://admin:secret@localhost:27017"   --db mydb   --collection orders   --type csv   --fields "orderId,customer.name,customer.email,shippingAddress.city,shippingAddress.country,total"   --out /tmp/orders-with-customer.csv
```

Output:

```text
orderId,customer.name,customer.email,shippingAddress.city,shippingAddress.country,total
ORD-001,Jane Smith,jane@example.com,New York,US,99.99
```

## Filtering Documents for CSV Export

```bash
# Export only completed orders as CSV
mongoexport   --uri "mongodb://admin:secret@localhost:27017"   --db mydb   --collection orders   --type csv   --fields "orderId,customerId,total,createdAt"   --query '{"status": "completed"}'   --out /tmp/completed-orders.csv

# Export orders above a certain value
mongoexport   --uri "mongodb://admin:secret@localhost:27017"   --db mydb   --collection orders   --type csv   --fields "orderId,customerId,total"   --query '{"total": {"$gte": 500}}'   --sort '{"total": -1}'   --out /tmp/high-value-orders.csv

# Export date-range filtered data
mongoexport   --uri "mongodb://admin:secret@localhost:27017"   --db mydb   --collection orders   --type csv   --fields "orderId,status,total,createdAt"   --query '{"createdAt": {"$gte": {"$date": "2026-01-01T00:00:00Z"}}}'   --out /tmp/orders-2026.csv
```

## Exporting Without Header Row

```bash
# Export CSV without header line (useful for appending to existing files)
mongoexport   --uri "mongodb://admin:secret@localhost:27017"   --db mydb   --collection orders   --type csv   --fields "orderId,status,total"   --noHeaderLine   --out /tmp/orders-no-header.csv
```

## Making CSV Excel-Compatible

MongoDB exports dates in ISO 8601 format and uses standard comma separation. For Excel:

```bash
# Export the data
mongoexport   --uri "mongodb://admin:secret@localhost:27017"   --db mydb   --collection orders   --type csv   --fields "orderId,customerId,status,total,createdAt"   --out /tmp/orders.csv

# Convert date format for Excel readability (using Python)
python3 << 'EOF'
import csv
from datetime import datetime

with open('/tmp/orders.csv') as infile,      open('/tmp/orders-excel.csv', 'w', newline='') as outfile:
    
    reader = csv.DictReader(infile)
    writer = csv.DictWriter(outfile, fieldnames=reader.fieldnames)
    writer.writeheader()
    
    for row in reader:
        # Convert ISO date to Excel-friendly format
        if row.get('createdAt'):
            try:
                dt = datetime.fromisoformat(row['createdAt'].replace('Z', '+00:00'))
                row['createdAt'] = dt.strftime('%Y-%m-%d %H:%M:%S')
            except ValueError:
                pass
        writer.writerow(row)

print("Excel-compatible CSV written to /tmp/orders-excel.csv")
EOF
```

## Handling Array Fields in CSV

Arrays are exported as-is (as a JSON array string within the CSV cell):

```text
orderId,tags,total
ORD-001,"[""electronics"",""premium""]",299.99
```

To flatten arrays, post-process with Python:

```python
import csv
import json

with open('/tmp/orders.csv') as infile,      open('/tmp/orders-flat.csv', 'w', newline='') as outfile:
    
    reader = csv.DictReader(infile)
    
    # Determine max tag count
    rows = list(reader)
    max_tags = max(
        len(json.loads(row['tags'])) if row.get('tags') else 0
        for row in rows
    )
    
    new_fields = ['orderId', 'total'] + [f'tag_{i+1}' for i in range(max_tags)]
    writer = csv.DictWriter(outfile, fieldnames=new_fields)
    writer.writeheader()
    
    for row in rows:
        new_row = {'orderId': row['orderId'], 'total': row['total']}
        tags = json.loads(row.get('tags', '[]'))
        for i, tag in enumerate(tags):
            new_row[f'tag_{i+1}'] = tag
        writer.writerow(new_row)
```

## Summary

`mongoexport` CSV mode requires specifying fields explicitly via `--fields` or `--fieldFile`, since MongoDB's flexible schema means fields vary between documents. Nested fields are accessible using dot notation. Combine `--query` for filtering, `--sort` for ordering, and `--limit` for capping output size. Post-process the CSV output with Python or shell tools to reformat dates, flatten arrays, or ensure Excel compatibility.
