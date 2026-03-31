# What Is MySQL Document Store

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Document Store, NoSQL, JSON, Collection

Description: MySQL Document Store lets you use MySQL as a NoSQL database by storing and querying JSON documents in collections alongside traditional relational tables.

---

## Overview

MySQL Document Store is a feature introduced in MySQL 5.7.12 that transforms MySQL into a hybrid database capable of handling both relational and document-based data. Rather than forcing all data into rigid row-and-column schemas, Document Store lets developers store flexible JSON documents in structures called collections.

This makes MySQL viable for applications that need schema flexibility without abandoning the reliability, ACID compliance, and tooling of a mature relational database.

## What Is a Collection

In MySQL Document Store, a collection is a special type of table that stores JSON documents. Each document is a JSON object, and each collection automatically gets a generated `_id` field used as the primary key.

Under the hood, a collection is a standard MySQL InnoDB table with a `doc` column of type `JSON` and an auto-generated `_id` indexed column. This means collections benefit from all standard MySQL storage engine features.

## Creating a Collection

Using MySQL Shell in JavaScript mode:

```javascript
var session = mysqlx.getSession("root:secret@localhost:33060");
var db = session.getSchema("myapp");

// Create a collection
db.createCollection("products");
```

Or using Python:

```python
import mysqlx

session = mysqlx.get_session("root:secret@localhost:33060")
db = session.get_schema("myapp")
db.create_collection("products")
```

## Adding Documents

```python
products = db.get_collection("products")

products.add(
    {"name": "Widget A", "price": 9.99, "tags": ["sale", "new"]},
    {"name": "Widget B", "price": 19.99, "tags": ["featured"]}
).execute()
```

## Querying Documents

```python
# Find all products under $15
result = products.find("price < 15").execute()
for doc in result.fetch_all():
    print(doc["name"], doc["price"])

# Find with field projection
result = products.find().fields("name", "price").execute()
```

## Modifying and Removing Documents

```python
# Update a field
products.modify("name = 'Widget A'").set("price", 7.99).execute()

# Remove documents
products.remove("price < 5").execute()
```

## Mixing Relational and Document Data

One of MySQL Document Store's most powerful aspects is coexistence with relational tables. You can join collection data with relational data using the SQL layer:

```sql
SELECT doc->>'$.name' AS product_name, o.quantity
FROM products, orders o
WHERE doc->>'$.sku' = o.sku;
```

## Indexing Collections

To improve query performance on document fields, create indexes on specific JSON paths:

```javascript
db.getCollection("products").createIndex("price_idx", {
    fields: [{field: "$.price", type: "DECIMAL(10,2)"}]
});
```

## Summary

MySQL Document Store provides a practical NoSQL experience within MySQL by enabling JSON document storage in collections, accessed through the X DevAPI. It is ideal for teams that want flexible document storage without giving up MySQL's transactional guarantees and operational maturity. Collections and relational tables coexist in the same database, making it easy to use each paradigm where it fits best.