# How to Use MySQL Shell in Python Mode

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Shell, Python, Database, Administration

Description: Learn how to switch MySQL Shell to Python mode and use Python scripting to automate database tasks, query data, and manage InnoDB clusters.

---

## Introduction

MySQL Shell is a powerful command-line interface that supports three modes: SQL, JavaScript, and Python. Python mode is especially useful for developers who prefer Python syntax and want to leverage Python libraries alongside MySQL operations. Switching to Python mode enables you to write scripts, automate administrative tasks, and interact with MySQL databases using familiar Python idioms.

## Switching to Python Mode

When you launch MySQL Shell, the default mode is JavaScript. To switch to Python mode, use the `\py` command:

```bash
mysqlsh
```

Once inside the shell:

```text
MySQL  JS > \py
Switching to Python mode...
MySQL  Py >
```

You can also start directly in Python mode from the command line:

```bash
mysqlsh --py
```

Or connect to a server and start in Python mode:

```bash
mysqlsh root@localhost --py
```

## Running SQL Queries from Python Mode

In Python mode, you can execute SQL statements using the `session.run_sql()` method:

```python
result = session.run_sql("SELECT user, host FROM mysql.user")
result.fetch_all()
```

You can also iterate over results:

```python
result = session.run_sql("SELECT * FROM employees.departments")
for row in result.fetch_all():
    print(row[0], row[1])
```

## Using the MySQL Shell X DevAPI

MySQL Shell's X DevAPI allows you to work with collections and tables using a document-store model. In Python mode:

```python
# Get a schema
db = session.get_schema("mydb")

# Get a collection
coll = db.get_collection("customers")

# Find documents
docs = coll.find("age > 30").execute()
for doc in docs.fetch_all():
    print(doc)
```

For relational tables, use the table API:

```python
table = db.get_table("orders")
result = table.select(["id", "total"]).where("total > 100").execute()
for row in result.fetch_all():
    print(row)
```

## Executing Python Scripts

You can run Python script files directly in MySQL Shell using the `\source` command or the `--file` flag:

```bash
mysqlsh root@localhost --py --file=/path/to/myscript.py
```

Inside a script, you can use the built-in `session` object to interact with the database:

```python
# myscript.py
result = session.run_sql("SHOW DATABASES")
for row in result.fetch_all():
    print(row[0])
```

## Using Python Mode for Automation

Python mode is especially useful for automating repetitive tasks. For example, you can write a loop that creates multiple test users:

```python
users = ["alice", "bob", "charlie"]
for user in users:
    session.run_sql(f"CREATE USER '{user}'@'localhost' IDENTIFIED BY 'TempPass1!'")
    session.run_sql(f"GRANT SELECT ON mydb.* TO '{user}'@'localhost'")
    print(f"User {user} created.")
```

## Accessing Shell Globals

MySQL Shell exposes several global objects in Python mode:

- `session` - the current database session
- `shell` - shell utility functions
- `dba` - InnoDB cluster management
- `util` - import/export utilities

```python
# Check the current session status
shell.status()

# Check connected user
print(session.run_sql("SELECT CURRENT_USER()").fetch_one()[0])
```

## Summary

MySQL Shell's Python mode gives you a flexible environment for scripting and automating MySQL operations using Python syntax. You can run SQL queries, use the X DevAPI for document and table access, execute Python scripts, and leverage shell globals like `dba` and `util` for advanced cluster and data management tasks.
