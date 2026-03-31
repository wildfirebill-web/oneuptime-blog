# How to Migrate from MySQL to MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, MySQL, Migration, Database, Schema

Description: Learn how to migrate data from MySQL to MongoDB by converting relational schemas to document models, exporting data, and transforming it for import.

---

## Overview

Migrating from MySQL to MongoDB involves more than just moving data - it requires rethinking your data model. Relational schemas normalize data across tables, while MongoDB encourages embedding related data in documents to reduce joins.

## Schema Design Conversion

Consider a MySQL schema with two tables:

```sql
CREATE TABLE customers (
  id INT PRIMARY KEY,
  name VARCHAR(255),
  email VARCHAR(255)
);

CREATE TABLE orders (
  id INT PRIMARY KEY,
  customer_id INT,
  total DECIMAL(10,2),
  status VARCHAR(50),
  FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

In MongoDB, you can embed the customer in each order or keep them separate. For a 1-to-many relationship, embedding the most-accessed fields reduces joins:

```javascript
{
  _id: ObjectId(),
  customer: { id: 1, name: "Alice", email: "alice@example.com" },
  total: 149.99,
  status: "completed"
}
```

## Step 1: Export Data from MySQL

Export each table to CSV:

```bash
mysql -u root -p mydb -e "SELECT * FROM customers" \
  --batch --silent > customers.csv

mysql -u root -p mydb -e "SELECT * FROM orders" \
  --batch --silent > orders.csv
```

Or use `mysqldump` to get SQL format for reference:

```bash
mysqldump -u root -p mydb > mydb_backup.sql
```

## Step 2: Transform Data with Python

Write a script to transform flat CSV rows into MongoDB documents:

```python
import csv
import json
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017/")
db = client["newdb"]

# Load customers into a dict for embedding
customers = {}
with open("customers.csv") as f:
    for row in csv.DictReader(f, delimiter="\t"):
        customers[row["id"]] = {
            "name": row["name"],
            "email": row["email"]
        }

# Transform orders and embed customer data
orders = []
with open("orders.csv") as f:
    for row in csv.DictReader(f, delimiter="\t"):
        orders.append({
            "mysqlId": int(row["id"]),
            "customer": customers.get(row["customer_id"], {}),
            "total": float(row["total"]),
            "status": row["status"]
        })

if orders:
    db.orders.insert_many(orders)
    print(f"Inserted {len(orders)} orders")
```

## Step 3: Verify the Migration

```javascript
// Check document count matches row count in MySQL
db.orders.countDocuments()

// Sample a few documents to verify data
db.orders.findOne({ status: "completed" })
```

## Step 4: Update Application Code

Replace SQL queries with MongoDB queries in your application:

```javascript
// MySQL
// SELECT * FROM orders WHERE status = 'completed' AND customer_id = 42

// MongoDB equivalent
db.orders.find({
  status: "completed",
  "customer.id": "42"
})
```

## Step 5: Run Both in Parallel

Before cutting over, run MySQL and MongoDB in parallel to verify results match. Use feature flags to gradually shift traffic.

## Summary

Migrating from MySQL to MongoDB requires converting normalized relational tables into document collections by embedding related data. Export MySQL data to CSV, transform it with a script that reflects your new document model, and import it into MongoDB. Run both systems in parallel and verify correctness before switching application traffic.
