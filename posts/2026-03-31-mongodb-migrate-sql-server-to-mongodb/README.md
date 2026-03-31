# How to Migrate from SQL Server to MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, SQL Server, Migration, Database, Schema

Description: A practical guide for migrating data and schema from Microsoft SQL Server to MongoDB, covering schema redesign, ETL transformation, and cutover strategies.

---

## Overview

Migrating from SQL Server to MongoDB requires both a data migration and a schema redesign. SQL Server stores data in normalized relational tables with strict types and foreign key constraints. MongoDB stores flexible JSON documents that can embed related data. This guide walks through the migration process step by step.

## Analyze and Redesign Your Schema

Start by identifying tables with one-to-many relationships that can be embedded in MongoDB documents.

```sql
-- SQL Server: typical normalized schema
CREATE TABLE Customers (Id INT PRIMARY KEY, Name NVARCHAR(100), Email NVARCHAR(200));
CREATE TABLE Orders (Id INT PRIMARY KEY, CustomerId INT REFERENCES Customers(Id), Total DECIMAL, Status NVARCHAR(20));
CREATE TABLE OrderItems (Id INT PRIMARY KEY, OrderId INT REFERENCES Orders(Id), ProductId INT, Qty INT, Price DECIMAL);
```

Redesign as MongoDB documents by embedding frequently accessed related data:

```javascript
// MongoDB: embedded document design
{
  _id: ObjectId(),
  customerId: 42,
  name: "Alice",
  email: "alice@example.com",
  orders: [
    {
      orderId: 101,
      total: 149.99,
      status: "completed",
      items: [
        { productId: 5, qty: 2, price: 29.99 },
        { productId: 8, qty: 1, price: 89.99 }
      ]
    }
  ]
}
```

## Export Data from SQL Server

Export tables to CSV using BCP (Bulk Copy Program) or SQL Server Management Studio (SSMS):

```bash
# BCP export from SQL Server to CSV
bcp "SELECT * FROM dbo.Customers" queryout customers.csv \
  -c -t "," -r "\n" \
  -S sqlserver-host -U sa -P yourpassword -d mydb
```

Or use SQL Server's Export Wizard in SSMS to export to flat files.

## Transform and Load with Python

Use `pyodbc` to query SQL Server directly and write transformed documents to MongoDB:

```python
import pyodbc
from pymongo import MongoClient, InsertOne

sql_conn = pyodbc.connect(
  "DRIVER={ODBC Driver 18 for SQL Server};SERVER=sql-host;DATABASE=mydb;UID=sa;PWD=pass"
)
mongo_client = MongoClient("mongodb://localhost:27017")
db = mongo_client["mydb"]

cursor = sql_conn.cursor()
cursor.execute("""
  SELECT c.Id, c.Name, c.Email,
         o.Id AS OrderId, o.Total, o.Status,
         oi.ProductId, oi.Qty, oi.Price
  FROM Customers c
  LEFT JOIN Orders o ON o.CustomerId = c.Id
  LEFT JOIN OrderItems oi ON oi.OrderId = o.Id
  ORDER BY c.Id, o.Id
""")

customers = {}
for row in cursor.fetchall():
  cid = row.Id
  if cid not in customers:
    customers[cid] = {"_id": cid, "name": row.Name, "email": row.Email, "orders": {}}
  if row.OrderId:
    oid = row.OrderId
    if oid not in customers[cid]["orders"]:
      customers[cid]["orders"][oid] = {"orderId": oid, "total": float(row.Total), "status": row.Status, "items": []}
    if row.ProductId:
      customers[cid]["orders"][oid]["items"].append({"productId": row.ProductId, "qty": row.Qty, "price": float(row.Price)})

# Flatten nested order dicts to lists
docs = []
for cust in customers.values():
  cust["orders"] = list(cust["orders"].values())
  docs.append(InsertOne(cust))

db.customers.bulk_write(docs, ordered=False)
print(f"Migrated {len(docs)} customers")
```

## Handle SQL Server Data Types

Some SQL Server types need explicit conversion:

```python
import decimal, datetime

def convert_value(v):
  if isinstance(v, decimal.Decimal):
    return float(v)
  if isinstance(v, (datetime.datetime, datetime.date)):
    return v  # MongoDB stores as BSON Date
  return v
```

## Validate Migration

```python
# Compare counts
cursor.execute("SELECT COUNT(*) FROM Customers")
sql_count = cursor.fetchone()[0]
mongo_count = db.customers.count_documents({})
print(f"SQL Server: {sql_count}, MongoDB: {mongo_count}")
assert sql_count == mongo_count
```

## Update Application Queries

```javascript
// Before (SQL Server T-SQL)
// SELECT * FROM Orders WHERE CustomerId = @id AND Status = 'pending'

// After (MongoDB)
db.customers.aggregate([
  { $match: { _id: customerId } },
  { $unwind: "$orders" },
  { $match: { "orders.status": "pending" } },
  { $replaceRoot: { newRoot: "$orders" } }
]);
```

## Summary

Migrating from SQL Server to MongoDB requires thoughtful schema redesign to collapse normalized relational tables into documents. Use BCP or pyodbc to export and transform data, validate record counts post-migration, and systematically replace T-SQL queries with MongoDB's aggregation pipeline. Run the migration in a staging environment first before cutting over production.
