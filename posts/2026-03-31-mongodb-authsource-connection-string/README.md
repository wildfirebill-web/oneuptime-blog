# How to Use the authSource Option in MongoDB Connection Strings

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Authentication, Connection String, Security, Configuration

Description: Learn how to use the authSource option in MongoDB connection strings to specify which database holds user credentials for authentication.

---

## What Is authSource?

By default, MongoDB authenticates users against the database specified in the connection string. The `authSource` option overrides this behavior and points the driver to a different database where the user's credentials are stored. This is especially important when users are defined in the `admin` database but need access to other databases.

## Basic Usage

```text
mongodb://myuser:mypassword@localhost:27017/myapp?authSource=admin
```

In this example, the connection targets the `myapp` database, but authentication is performed against the `admin` database where `myuser` is defined.

## When Do You Need authSource?

Most MongoDB deployments create application users in the `admin` database to keep user management centralized. Without `authSource=admin`, the driver attempts to authenticate against `myapp`, fails, and throws an `Authentication failed` error.

```text
# User defined in admin database
use admin
db.createUser({
  user: "appuser",
  pwd: "secret",
  roles: [{ role: "readWrite", db: "myapp" }]
})
```

```text
# Correct connection string
mongodb://appuser:secret@localhost:27017/myapp?authSource=admin

# Incorrect - will fail if appuser is not in myapp
mongodb://appuser:secret@localhost:27017/myapp
```

## Node.js Driver

```javascript
const { MongoClient } = require('mongodb');

const client = new MongoClient(
  'mongodb://appuser:secret@localhost:27017/myapp',
  { authSource: 'admin' }
);

// Or via the connection string
const client2 = new MongoClient(
  'mongodb://appuser:secret@localhost:27017/myapp?authSource=admin'
);

await client.connect();
const db = client.db('myapp');
const users = await db.collection('users').find().toArray();
```

## PyMongo

```python
from pymongo import MongoClient

# Option 1: via connection string
client = MongoClient("mongodb://appuser:secret@localhost:27017/myapp?authSource=admin")

# Option 2: via keyword argument
client = MongoClient(
    host="localhost",
    port=27017,
    username="appuser",
    password="secret",
    authSource="admin",
    authMechanism="SCRAM-SHA-256",
)

db = client["myapp"]
```

## Using authSource with LDAP

For LDAP authentication, the `authSource` must be set to `$external`:

```text
mongodb://ldapuser:password@host:27017/myapp?authSource=$external&authMechanism=PLAIN
```

```javascript
const client = new MongoClient('mongodb://host:27017', {
  auth: { username: 'cn=user,dc=example,dc=com', password: 'ldappassword' },
  authSource: '$external',
  authMechanism: 'PLAIN',
});
```

## Verifying the authSource

You can confirm where a user is stored using mongosh:

```bash
mongosh --eval "use admin; db.getUsers()" mongodb://localhost:27017
```

## Common Error

```text
MongoServerError: Authentication failed.
```

This often means `authSource` is missing or pointing to the wrong database. Double-check that the user exists in the database specified by `authSource`.

## Summary

The `authSource` parameter is essential when your MongoDB users are stored in a different database than the one you are connecting to - most commonly `admin`. Always specify `authSource=admin` when your application user was created in the admin database, and use `authSource=$external` for LDAP or x.509 authentication.
