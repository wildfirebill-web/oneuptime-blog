# How to Connect to MySQL from Python

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Python, Connection, Driver, Database

Description: Learn how to connect to a MySQL database from Python using the most popular libraries including mysql-connector-python, PyMySQL, and SQLAlchemy.

---

## Overview

Python has several mature libraries for connecting to MySQL. The three most commonly used are:

| Library | Type | Best For |
|---------|------|----------|
| `mysql-connector-python` | Official Oracle driver | General use, pure Python |
| `PyMySQL` | Pure Python | Environments where C extensions are unavailable |
| `SQLAlchemy` | ORM + Core | Complex applications, ORM patterns |

## Installing a Driver

```bash
# Official MySQL Connector
pip install mysql-connector-python

# PyMySQL
pip install PyMySQL

# SQLAlchemy with PyMySQL
pip install sqlalchemy pymysql
```

## Connecting with mysql-connector-python

```python
import mysql.connector

conn = mysql.connector.connect(
    host='localhost',
    port=3306,
    user='app_user',
    password='secret',
    database='shop',
    charset='utf8mb4',
    collation='utf8mb4_unicode_ci'
)

cursor = conn.cursor()
cursor.execute("SELECT VERSION()")
version = cursor.fetchone()
print(f"MySQL version: {version[0]}")

cursor.close()
conn.close()
```

## Connecting with PyMySQL

```python
import pymysql

conn = pymysql.connect(
    host='localhost',
    port=3306,
    user='app_user',
    password='secret',
    database='shop',
    charset='utf8mb4',
    cursorclass=pymysql.cursors.DictCursor
)

with conn:
    with conn.cursor() as cursor:
        cursor.execute("SELECT id, name FROM products LIMIT 5")
        rows = cursor.fetchall()
        for row in rows:
            print(row)
```

## Connecting with SQLAlchemy

```python
from sqlalchemy import create_engine, text

engine = create_engine(
    "mysql+pymysql://app_user:secret@localhost:3306/shop"
    "?charset=utf8mb4",
    pool_pre_ping=True,
    pool_size=5,
    max_overflow=10
)

with engine.connect() as conn:
    result = conn.execute(text("SELECT id, name FROM products LIMIT 5"))
    for row in result:
        print(row)
```

## Using Environment Variables for Credentials

Never hardcode credentials. Use environment variables:

```python
import os
import mysql.connector

conn = mysql.connector.connect(
    host=os.environ['DB_HOST'],
    user=os.environ['DB_USER'],
    password=os.environ['DB_PASSWORD'],
    database=os.environ['DB_NAME'],
    charset='utf8mb4'
)
```

## Executing Parameterized Queries

Always use parameterized queries to prevent SQL injection:

```python
cursor = conn.cursor()
cursor.execute(
    "SELECT id, email FROM users WHERE email = %s",
    ("alice@example.com",)
)
user = cursor.fetchone()
```

## Summary

Python offers multiple solid options for MySQL connectivity. Use `mysql-connector-python` for official support, `PyMySQL` for pure-Python deployments, and `SQLAlchemy` when you need an ORM or connection pooling. Always set `charset=utf8mb4`, use parameterized queries, and manage connection lifecycle with context managers or explicit `close()` calls.
