# How to Use MySQL Shell with X DevAPI in Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, MySQL Shell, Python, X DevAPI, Script

Description: Learn how to use MySQL Shell in Python mode to interact with MySQL using the X DevAPI for document collections and relational tables.

---

## What is MySQL Shell Python Mode?

MySQL Shell supports Python as one of its three interactive modes. In Python mode, the built-in `session` and `mysqlx` objects are available, giving you full access to the X DevAPI for document store and table operations. Python mode is especially useful for writing administrative scripts and data exploration tasks familiar to Python developers.

## Connecting in Python Mode

```bash
mysqlsh --uri mysqlx://root@127.0.0.1:33060 --py
```

Or connect first and then switch:

```bash
mysqlsh --uri mysqlx://root@127.0.0.1:33060
\py
```

## Exploring Schemas and Collections

```python
# List schemas
schemas = session.get_schemas()
for s in schemas:
    print(s.get_name())

# Get a specific schema
db = session.get_schema('mydb')

# List collections
for col in db.get_collections():
    print('Collection:', col.get_name())

# List tables
for tbl in db.get_tables():
    print('Table:', tbl.get_name())
```

Note: Python mode uses `snake_case` method names instead of JavaScript's `camelCase`.

## Collection CRUD in Python

```python
products = db.get_collection('products')

# Add documents
products.add(
    {'name': 'Python Primer', 'price': 29.99, 'category': 'books', 'inStock': True},
    {'name': 'Data Handbook', 'price': 49.99, 'category': 'books', 'inStock': False}
).execute()

# Find documents
result = products.find('inStock = true').execute()
for doc in result.fetch_all():
    print(doc)

# Modify documents
products.modify('name = :n').set('price', 32.99).bind('n', 'Python Primer').execute()

# Remove documents
products.remove('inStock = false').execute()
```

## Running SQL from Python Mode

```python
# Execute raw SQL
result = session.sql('SELECT VERSION()').execute()
print(result.fetch_one()[0])

# Use a database
session.sql('USE mydb').execute()

# Iterate query results
result = session.sql('SELECT id, name, email FROM customers LIMIT 5').execute()
for row in result.fetch_all():
    print(row[0], row[1], row[2])
```

## Table Operations in Python Mode

```python
customers = db.get_table('customers')

# Insert
customers.insert('name', 'email', 'tier').values(
    'Diana', 'diana@example.com', 'gold'
).execute()

# Select
result = customers.select('id', 'name', 'tier').where('tier = :t').bind('t', 'gold').execute()
for row in result.fetch_all():
    print(row)

# Update
customers.update().set('tier', 'platinum').where('id = :id').bind('id', 5).execute()

# Delete
customers.delete().where('tier = :t').bind('t', 'expired').execute()
```

## Running a Python Script File

Create `batch_insert.py`:

```python
db = session.get_schema('mydb')
col = db.get_collection('events')

for i in range(100):
    col.add({'eventId': i, 'type': 'test', 'value': i * 1.5}).execute()

print('Inserted 100 events')
```

Run it:

```bash
mysqlsh --uri mysqlx://root:secret@127.0.0.1:33060 --file batch_insert.py
```

## Summary

MySQL Shell in Python mode exposes the full X DevAPI with Pythonic snake_case method names. Use it for interactive data exploration, quick administrative scripts, and testing X DevAPI patterns before implementing them in application connectors. Combine collection CRUD with `session.sql()` for SQL access when you need joins or aggregations that fall outside the X DevAPI's native capabilities.
