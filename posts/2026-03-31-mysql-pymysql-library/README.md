# How to Use MySQL with Python's PyMySQL Library

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Python, PyMySQL, Driver, Query

Description: A practical guide to using the PyMySQL library to connect, query, and manage MySQL databases from Python applications.

---

## Why PyMySQL?

PyMySQL is a pure-Python MySQL client that requires no compiled C extensions. This makes it ideal for environments where binary packages are restricted (such as certain cloud functions or minimal container images) and for drop-in replacement of `MySQLdb` in Python 3 projects.

## Installation

```bash
pip install PyMySQL
```

## Establishing a Connection

```python
import pymysql

conn = pymysql.connect(
    host='localhost',
    port=3306,
    user='app_user',
    password='secret',
    database='shop',
    charset='utf8mb4',
    autocommit=False
)
```

## Using DictCursor for Dictionary Results

By default, PyMySQL returns rows as tuples. `DictCursor` returns dictionaries keyed by column name:

```python
import pymysql
import pymysql.cursors

conn = pymysql.connect(
    host='localhost',
    user='app_user',
    password='secret',
    database='shop',
    charset='utf8mb4',
    cursorclass=pymysql.cursors.DictCursor
)

with conn.cursor() as cursor:
    cursor.execute("SELECT id, name, price FROM products WHERE price < %s", (50.00,))
    products = cursor.fetchall()
    for p in products:
        print(f"{p['name']}: ${p['price']}")
```

## Inserting Data and Committing Transactions

PyMySQL does not autocommit by default (when `autocommit=False`). Explicitly commit after write operations:

```python
with conn.cursor() as cursor:
    cursor.execute(
        "INSERT INTO orders (customer_id, total) VALUES (%s, %s)",
        (42, 99.99)
    )
    order_id = cursor.lastrowid
conn.commit()
print(f"Inserted order ID: {order_id}")
```

Roll back on errors:

```python
try:
    with conn.cursor() as cursor:
        cursor.execute("UPDATE inventory SET stock = stock - 1 WHERE product_id = %s", (7,))
    conn.commit()
except Exception as e:
    conn.rollback()
    raise e
```

## Fetching Results

```python
with conn.cursor() as cursor:
    cursor.execute("SELECT * FROM categories")

    # Fetch one row
    row = cursor.fetchone()

    # Fetch all remaining rows
    rows = cursor.fetchall()

    # Fetch N rows at a time
    chunk = cursor.fetchmany(size=100)
```

## Using as a MySQLdb Drop-In Replacement

Projects that import `MySQLdb` can use PyMySQL without code changes:

```python
import pymysql
pymysql.install_as_MySQLdb()

import MySQLdb  # Now backed by PyMySQL
```

## Closing the Connection

```python
conn.close()
```

Or use a context manager to ensure cleanup:

```python
with pymysql.connect(host='localhost', user='app_user',
                     password='secret', database='shop') as conn:
    with conn.cursor() as cursor:
        cursor.execute("SELECT COUNT(*) FROM orders")
        count = cursor.fetchone()[0]
print(f"Total orders: {count}")
```

## Summary

PyMySQL is a lightweight, pure-Python MySQL driver well suited for Python 3 applications that cannot use C extensions. Use `DictCursor` for readable results, always use parameterized queries with `%s` placeholders, and manage transactions explicitly with `commit()` and `rollback()`.
