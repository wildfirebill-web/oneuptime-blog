# How to Get All Databases in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Database, Administration, Query

Description: Learn how to list all databases in MongoDB using the shell, drivers, and admin commands with practical examples for ops and developers.

---

## Listing Databases in the MongoDB Shell

The simplest way to see all databases is to open `mongosh` and run the built-in command:

```javascript
show dbs
```

This returns each database name alongside its on-disk size. The output looks like:

```text
admin    180.00 KiB
config    72.00 KiB
local     40.00 KiB
myapp    512.00 KiB
```

Note that `show dbs` only lists databases that contain at least one collection with data. An empty database you just created will not appear here until you insert a document.

## Using the listDatabases Admin Command

For scripting or when you need more metadata, use the `listDatabases` command against the `admin` database:

```javascript
db.adminCommand({ listDatabases: 1 })
```

The response is a document with a `databases` array and a total `totalSize` field:

```json
{
  "databases": [
    { "name": "admin",  "sizeOnDisk": 184320, "empty": false },
    { "name": "config", "sizeOnDisk": 73728,  "empty": false },
    { "name": "local",  "sizeOnDisk": 40960,  "empty": false },
    { "name": "myapp",  "sizeOnDisk": 524288, "empty": false }
  ],
  "totalSize": 823296,
  "ok": 1
}
```

To get only the names as an array, pass `nameOnly: true` - this is much faster because MongoDB skips calculating sizes:

```javascript
db.adminCommand({ listDatabases: 1, nameOnly: true })
```

You can also filter by name using a regex with the `filter` option:

```javascript
db.adminCommand({ listDatabases: 1, nameOnly: true, filter: { name: /^app/ } })
```

## Listing Databases from Application Drivers

### Node.js (Official Driver)

```javascript
const { MongoClient } = require("mongodb");

async function listAllDatabases() {
  const client = new MongoClient("mongodb://localhost:27017");
  await client.connect();

  const adminDb = client.db("admin");
  const result = await adminDb.command({ listDatabases: 1, nameOnly: true });

  result.databases.forEach((db) => console.log(db.name));

  await client.close();
}

listAllDatabases();
```

### Python (PyMongo)

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017")
databases = client.list_database_names()

for name in databases:
    print(name)
```

`list_database_names()` is a convenience wrapper around `listDatabases` that returns a plain list of strings.

## Filtering Out System Databases

In automation scripts you typically want to skip `admin`, `config`, and `local`. Use a simple exclusion list:

```python
SYSTEM_DBS = {"admin", "config", "local"}

user_databases = [
    name for name in client.list_database_names()
    if name not in SYSTEM_DBS
]
```

Or in the shell:

```javascript
db.adminCommand({ listDatabases: 1, nameOnly: true }).databases
  .map(d => d.name)
  .filter(n => !["admin", "config", "local"].includes(n));
```

## Required Permissions

The `listDatabases` command requires the `listDatabases` privilege on the cluster resource. The built-in `readAnyDatabase` or `dbAdminAnyDatabase` roles cover it. When auth is enabled and the user lacks cluster-wide privileges, MongoDB returns only the databases that user has access to, so the list may be incomplete.

## Summary

Use `show dbs` for quick interactive inspection and `db.adminCommand({ listDatabases: 1 })` when you need metadata or are scripting. Pass `nameOnly: true` to avoid an expensive size calculation, and use the `filter` option to narrow results by database name. In application code, the driver convenience methods (`list_database_names()` in PyMongo, `adminDb.command()` in Node.js) wrap the same underlying command cleanly.
