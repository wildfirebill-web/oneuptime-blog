# How to Export MySQL Data to JSON

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, JSON, Export, JSON Function, Data Migration

Description: Learn how to export MySQL query results to JSON format using built-in JSON functions, the mysql client, and Python scripts for downstream consumption.

---

## Introduction

Exporting MySQL data as JSON is useful for feeding APIs, transferring data to document stores, or creating data snapshots. MySQL 5.7+ provides native JSON aggregation functions that let you build JSON output directly in SQL, eliminating the need for application-level serialization in many cases.

## Exporting Rows as a JSON Array with JSON_ARRAYAGG

Use `JSON_ARRAYAGG` combined with `JSON_OBJECT` to export rows as a JSON array:

```sql
SELECT JSON_ARRAYAGG(
  JSON_OBJECT(
    'id', id,
    'name', name,
    'email', email,
    'created_at', created_at
  )
) AS json_export
FROM customers;
```

Write the result to a file using `SELECT INTO OUTFILE`:

```sql
SELECT JSON_ARRAYAGG(
  JSON_OBJECT('id', id, 'name', name, 'email', email)
)
FROM customers
INTO OUTFILE '/tmp/customers.json'
LINES TERMINATED BY '';
```

## Exporting One JSON Object Per Row

For NDJSON (newline-delimited JSON), emit one JSON object per line:

```sql
SELECT JSON_OBJECT('id', id, 'name', name, 'email', email)
FROM customers
INTO OUTFILE '/tmp/customers.ndjson'
LINES TERMINATED BY '\n';
```

## Using the mysql Client for Client-Side Export

For exporting to a local file without server filesystem access:

```bash
mysql -u root -p mydb \
  -e "SELECT JSON_ARRAYAGG(JSON_OBJECT('id',id,'name',name,'email',email)) FROM customers" \
  --silent --raw > customers.json
```

## Exporting with Python

For more control over formatting and filtering:

```python
import json
import mysql.connector

conn = mysql.connector.connect(
    host='localhost',
    user='root',
    password='password',
    database='mydb'
)
cursor = conn.cursor(dictionary=True)
cursor.execute("SELECT id, name, email, created_at FROM customers")
rows = cursor.fetchall()

# Convert date objects to strings for JSON serialization
for row in rows:
    if row.get('created_at'):
        row['created_at'] = str(row['created_at'])

with open('customers.json', 'w') as f:
    json.dump(rows, f, indent=2)

print(f"Exported {len(rows)} rows")
cursor.close()
conn.close()
```

## Exporting Nested JSON Structures

Export orders with their line items as nested JSON:

```sql
SELECT JSON_OBJECT(
  'order_id', o.id,
  'customer', o.customer_name,
  'items', (
    SELECT JSON_ARRAYAGG(
      JSON_OBJECT('product', i.product_name, 'qty', i.quantity)
    )
    FROM order_items i WHERE i.order_id = o.id
  )
)
FROM orders o
INTO OUTFILE '/tmp/orders_nested.ndjson'
LINES TERMINATED BY '\n';
```

## Streaming Large Exports

For tables too large to fit in memory, stream rows in chunks:

```python
import json
import mysql.connector

conn = mysql.connector.connect(
    host='localhost', user='root',
    password='password', database='mydb'
)
cursor = conn.cursor(dictionary=True)
cursor.execute("SELECT id, name, email FROM customers")

with open('customers_stream.ndjson', 'w') as f:
    for row in cursor:
        f.write(json.dumps(row) + '\n')

cursor.close()
conn.close()
```

## Summary

MySQL's `JSON_OBJECT` and `JSON_ARRAYAGG` functions make it possible to export relational data as JSON entirely within SQL. For larger or more complex exports, Python with cursor iteration provides fine-grained control over formatting, date serialization, and streaming. Choose NDJSON for large datasets that need to be processed line by line, and JSON arrays for smaller exports consumed by APIs.
