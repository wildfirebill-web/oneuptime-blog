# How to Connect to MongoDB with Authentication via Connection String

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Authentication, Connection, Security, Driver

Description: Learn how to connect to MongoDB with username and password authentication via connection string in Node.js, Python, Java, and the mongosh shell.

---

Authentication is a critical security requirement for any MongoDB deployment. The connection string URI embeds credentials and authentication parameters, making it the standard way to configure authenticated connections across all official drivers.

## Authentication URI Structure

```text
mongodb://username:password@host:port/database?authSource=authDB&authMechanism=SCRAM-SHA-256
```

Key parameters:
- `authSource` - the database where the user account is defined (usually `admin`)
- `authMechanism` - the authentication protocol (defaults to SCRAM-SHA-256 in MongoDB 4.0+)

## Connecting via mongosh

```bash
mongosh "mongodb://appUser:myPassword@localhost:27017/myDatabase?authSource=admin"
```

Or using separate flags:

```bash
mongosh --host localhost --port 27017 \
  --username appUser \
  --password myPassword \
  --authenticationDatabase admin
```

## Node.js with Mongoose

```javascript
const mongoose = require("mongoose");

const uri = "mongodb://appUser:myPassword@localhost:27017/myDatabase?authSource=admin";

mongoose.connect(uri, {
  serverSelectionTimeoutMS: 10000
})
.then(() => console.log("Connected"))
.catch(err => console.error("Connection error:", err));
```

Using the native driver:

```javascript
const { MongoClient } = require("mongodb");

const client = new MongoClient(
  "mongodb://appUser:myPassword@localhost:27017/?authSource=admin",
  { serverSelectionTimeoutMS: 10000 }
);

async function connect() {
  await client.connect();
  const db = client.db("myDatabase");
  await db.command({ ping: 1 });
  console.log("Authenticated successfully");
}

connect().catch(console.error);
```

## Python with PyMongo

```python
from pymongo import MongoClient
from pymongo.errors import ConnectionFailure

uri = "mongodb://appUser:myPassword@localhost:27017/?authSource=admin"
client = MongoClient(uri, serverSelectionTimeoutMS=10000)

try:
    client.admin.command("ping")
    print("Authenticated successfully")
except ConnectionFailure as e:
    print(f"Connection failed: {e}")
```

## Java with the MongoDB Driver

```java
import com.mongodb.client.MongoClients;
import com.mongodb.client.MongoClient;

public class App {
    public static void main(String[] args) {
        String uri = "mongodb://appUser:myPassword@localhost:27017/?authSource=admin";
        try (MongoClient mongoClient = MongoClients.create(uri)) {
            mongoClient.getDatabase("admin")
                .runCommand(new org.bson.Document("ping", 1));
            System.out.println("Authenticated successfully");
        }
    }
}
```

## Handling Special Characters in Passwords

Passwords with special characters must be percent-encoded in the URI. Use a helper:

```javascript
// Node.js
const username = encodeURIComponent("app@user");
const password = encodeURIComponent("p@ss:word!");
const uri = `mongodb://${username}:${password}@localhost:27017/?authSource=admin`;
```

```python
# Python
from urllib.parse import quote_plus
uri = f"mongodb://{quote_plus('app@user')}:{quote_plus('p@ss:word!')}@localhost:27017/?authSource=admin"
```

## Environment Variables for Credentials

Never hardcode credentials. Use environment variables:

```bash
export MONGO_URI="mongodb://appUser:myPassword@localhost:27017/myDatabase?authSource=admin"
```

```javascript
const uri = process.env.MONGO_URI;
const client = new MongoClient(uri);
```

```python
import os
from pymongo import MongoClient
client = MongoClient(os.environ["MONGO_URI"])
```

## Verifying the Authentication Mechanism

```javascript
db.runCommand({ connectionStatus: 1 })
```

The output shows the authenticated user and roles:

```json
{
  "authInfo": {
    "authenticatedUsers": [{ "user": "appUser", "db": "admin" }],
    "authenticatedUserRoles": [{ "role": "readWrite", "db": "myDatabase" }]
  }
}
```

## Summary

Embed credentials in the MongoDB URI using `username:password@host/db?authSource=admin`. Always percent-encode special characters in credentials, store URIs in environment variables rather than source code, and use `connectionStatus` to verify authentication succeeded after connecting.
