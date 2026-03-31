# What Is the MySQL X DevAPI

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, X DevAPI, Document Store, NoSQL, Database

Description: The MySQL X DevAPI is a modern developer API for MySQL that supports both relational and document-based data access using a fluent, object-oriented interface.

---

## Overview

The MySQL X DevAPI is a next-generation API for MySQL that enables developers to interact with MySQL using a modern, object-oriented interface. Unlike the traditional MySQL client API, the X DevAPI supports both relational SQL access and document-based (NoSQL) access through the same unified interface. It was introduced alongside MySQL 5.7.12 as part of the MySQL Shell and MySQL Connectors update.

The X DevAPI communicates over the MySQL X Protocol, which runs on port 33060 by default - separate from the classic MySQL protocol on port 3306.

## Key Concepts

The X DevAPI introduces several core abstractions:

- **Session** - a connection to the MySQL server
- **Schema** - a logical grouping of collections and tables
- **Collection** - a set of JSON documents (NoSQL access)
- **Table** - relational table access through the API

This design allows the same codebase to handle both structured relational data and flexible JSON document storage.

## Connecting with the X DevAPI

Using MySQL Connector/Python with the X DevAPI:

```python
import mysqlx

session = mysqlx.get_session({
    "host": "localhost",
    "port": 33060,
    "user": "root",
    "password": "secret"
})

schema = session.get_schema("mydb")
collection = schema.get_collection("users")
```

The connection string format is also supported:

```python
session = mysqlx.get_session("root:secret@localhost:33060")
```

## Working with Collections (NoSQL)

Adding and querying JSON documents:

```python
collection = schema.get_collection("users")

# Add a document
collection.add({
    "name": "Alice",
    "email": "alice@example.com",
    "age": 30
}).execute()

# Find documents
result = collection.find("age > 25").execute()
for doc in result.fetch_all():
    print(doc["name"])
```

## Working with Tables (Relational)

The X DevAPI also supports relational table CRUD:

```python
table = schema.get_table("orders")

# Select rows
result = table.select("id", "total").where("total > 100").execute()
for row in result.fetch_all():
    print(row["id"], row["total"])

# Insert rows
table.insert("customer_id", "total").values(42, 250.00).execute()
```

## Running SQL with the X DevAPI

You can still execute raw SQL through the X DevAPI session:

```python
result = session.sql("SELECT version()").execute()
row = result.fetch_one()
print(row[0])
```

This is useful when you need fine-grained SQL control but still want the X DevAPI session management.

## X DevAPI vs Classic API

| Feature | Classic API | X DevAPI |
|---|---|---|
| Protocol | MySQL Protocol (3306) | X Protocol (33060) |
| NoSQL support | No | Yes (Collections) |
| Interface style | Procedural | Object-oriented / fluent |
| CRUD operations | SQL only | Native CRUD methods |

## Summary

The MySQL X DevAPI provides a modern, unified interface for accessing MySQL as both a relational database and a document store. It is available across multiple language connectors including Python, Node.js, Java, and C++. By abstracting over both SQL and NoSQL paradigms, the X DevAPI simplifies development for applications that need flexible data access patterns.