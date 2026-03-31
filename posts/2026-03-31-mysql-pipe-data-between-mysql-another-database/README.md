# How to Pipe Data Between MySQL and Another Database

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Data Pipeline, Migration, PostgreSQL, ETL

Description: Learn how to pipe data between MySQL and other databases like PostgreSQL, SQLite, and MongoDB using command-line tools, Python, and ETL utilities.

---

## Introduction

Moving data between MySQL and heterogeneous databases (PostgreSQL, SQLite, MongoDB, etc.) is a common ETL task. Since different databases have incompatible dump formats, you typically extract data as CSV or use a Python/ETL tool to bridge the gap.

## MySQL to PostgreSQL via CSV

Export from MySQL and import into PostgreSQL:

```bash
# Step 1: Export from MySQL to CSV
mysql -u root -p source_db \
  --batch -e "SELECT id, name, email FROM users" \
  | sed 's/\t/,/g' > /tmp/users.csv

# Step 2: Create table in PostgreSQL
psql -U postgres target_db -c "
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(150)
);"

# Step 3: Import into PostgreSQL
psql -U postgres target_db -c "
\COPY users FROM '/tmp/users.csv' CSV HEADER;"
```

## MySQL to PostgreSQL via Python

For a more robust pipeline with type handling:

```python
import mysql.connector
import psycopg2

mysql_conn = mysql.connector.connect(
    host='mysql-host', user='root',
    password='password', database='source_db'
)
pg_conn = psycopg2.connect(
    host='pg-host', user='postgres',
    password='password', dbname='target_db'
)

mysql_cursor = mysql_conn.cursor()
pg_cursor = pg_conn.cursor()

mysql_cursor.execute("SELECT id, name, email, created_at FROM users")

batch = []
for row in mysql_cursor:
    batch.append(row)
    if len(batch) == 1000:
        pg_cursor.executemany(
            "INSERT INTO users (id, name, email, created_at) VALUES (%s, %s, %s, %s) ON CONFLICT DO NOTHING",
            batch
        )
        pg_conn.commit()
        batch.clear()

if batch:
    pg_cursor.executemany(
        "INSERT INTO users (id, name, email, created_at) VALUES (%s, %s, %s, %s) ON CONFLICT DO NOTHING",
        batch
    )
    pg_conn.commit()

print("Transfer complete")
mysql_cursor.close()
pg_cursor.close()
mysql_conn.close()
pg_conn.close()
```

## MySQL to SQLite

```bash
# Export MySQL table as SQL
mysqldump --compatible=ansi --skip-add-drop-table \
  --no-create-info -u root -p source_db users > users_data.sql

# Or pipe via Python for clean SQLite import
python3 << 'EOF'
import mysql.connector
import sqlite3

mysql_conn = mysql.connector.connect(
    host='localhost', user='root', password='password', database='source_db'
)
sqlite_conn = sqlite3.connect('output.db')

mysql_cur = mysql_conn.cursor()
mysql_cur.execute("SELECT id, name, email FROM users")

sqlite_conn.execute("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, name TEXT, email TEXT)")
sqlite_conn.executemany("INSERT OR REPLACE INTO users VALUES (?,?,?)", mysql_cur)
sqlite_conn.commit()
sqlite_conn.close()
mysql_conn.close()
EOF
```

## MySQL to MongoDB

```python
import mysql.connector
from pymongo import MongoClient

mysql_conn = mysql.connector.connect(
    host='localhost', user='root',
    password='password', database='source_db'
)
mongo_client = MongoClient('mongodb://localhost:27017/')
db = mongo_client['target_db']
collection = db['users']

cursor = mysql_conn.cursor(dictionary=True)
cursor.execute("SELECT id, name, email, metadata FROM users")

batch = list(cursor.fetchmany(1000))
while batch:
    collection.insert_many(batch)
    batch = list(cursor.fetchmany(1000))

print("Migrated to MongoDB")
cursor.close()
mysql_conn.close()
```

## Using pgloader for Automated MySQL to PostgreSQL Migration

`pgloader` automates schema conversion and data migration:

```bash
sudo apt install pgloader

pgloader mysql://root:password@localhost/source_db \
         pgsql://postgres:password@localhost/target_db
```

pgloader handles type mapping, index creation, and constraint conversion automatically.

## Summary

Piping data between MySQL and other databases works best with CSV as a neutral intermediate format for simple cases, or Python for type-safe bidirectional transfers. For full MySQL-to-PostgreSQL migrations, `pgloader` automates schema conversion. Always transfer in batches to avoid memory exhaustion and verify row counts in the destination after transfer.
