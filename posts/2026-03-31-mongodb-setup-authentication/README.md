# How to Set Up Authentication in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Authentication, Security, Operation, Administration

Description: Step-by-step guide to enabling MongoDB authentication, creating the first admin user, and connecting with credentials using mongosh and application drivers.

---

## How MongoDB Authentication Works

By default, MongoDB does not require authentication - any client that can reach the port can read and write data. Enabling authentication forces every client to provide a username and password (or certificate) before performing any operations.

MongoDB uses SCRAM (Salted Challenge Response Authentication Mechanism) as the default authentication mechanism. SCRAM-SHA-256 is used for new users unless the client only supports SCRAM-SHA-1.

```mermaid
sequenceDiagram
    participant Client
    participant MongoDB
    Client->>MongoDB: Connect (no credentials)
    MongoDB-->>Client: Authentication required
    Client->>MongoDB: authenticate(user, password)
    MongoDB->>MongoDB: Verify SCRAM-SHA-256 challenge
    MongoDB-->>Client: Authentication success
    Client->>MongoDB: db.collection.find()
    MongoDB-->>Client: Results
```

## Step 1 - Start MongoDB Without Authentication

When setting up authentication for the first time, you must create the initial admin user before enabling authentication. Start MongoDB without auth (or connect via localhost exception if freshly installed):

```bash
sudo systemctl start mongod
```

If `mongod.conf` has `authorization: disabled` (the default), connect via mongosh:

```bash
mongosh
```

## Step 2 - Create the Admin User

Switch to the `admin` database and create a user with the `userAdminAnyDatabase` role. This role can create and manage all users but cannot read or write regular data.

```javascript
use admin

db.createUser({
  user: "adminUser",
  pwd: passwordPrompt(),   // prompts securely, never hardcode passwords
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" }
  ]
})
```

Expected output:

```text
{ ok: 1 }
```

## Step 3 - Enable Authentication in mongod.conf

Open `/etc/mongod.conf` and add the security section:

```yaml
security:
  authorization: enabled
```

Restart MongoDB:

```bash
sudo systemctl restart mongod
```

## Step 4 - Verify Authentication is Required

Without credentials, commands should now fail:

```bash
mongosh --eval "db.adminCommand({ listDatabases: 1 })"
```

Expected error:

```text
MongoServerError: Command listDatabases requires authentication
```

## Step 5 - Connect with Credentials

Authenticate using mongosh with a connection string:

```bash
mongosh "mongodb://adminUser:yourPassword@127.0.0.1:27017/?authSource=admin"
```

Or connect and then authenticate interactively:

```bash
mongosh
```

```javascript
use admin
db.auth("adminUser", passwordPrompt())
```

## Creating Application-Specific Users

Create a separate user for each application with only the permissions it needs. Do not use the admin user in application code.

```javascript
use myapp

db.createUser({
  user: "appUser",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
})
```

Verify the user was created:

```javascript
db.getUser("appUser")
```

## Connecting from Node.js with Authentication

```javascript
const { MongoClient } = require("mongodb");

const uri = "mongodb://appUser:yourPassword@127.0.0.1:27017/myapp?authSource=myapp";

async function main() {
  const client = new MongoClient(uri);
  await client.connect();
  const db = client.db("myapp");
  const result = await db.collection("items").findOne({});
  console.log(result);
  await client.close();
}

main().catch(console.error);
```

## Connecting from Python with Authentication

```python
from pymongo import MongoClient

client = MongoClient(
    "mongodb://appUser:yourPassword@127.0.0.1:27017/myapp",
    authSource="myapp"
)

db = client["myapp"]
result = db["items"].find_one({})
print(result)
```

## The Localhost Exception

When `authorization: enabled` and no users exist in the `admin` database, MongoDB allows a single unauthenticated connection from localhost to create the first user. This is called the localhost exception. Once the first user is created, the exception is disabled automatically.

If you lose the admin password on a standalone instance, you can temporarily disable authentication, create a new admin user, and re-enable it.

## Changing a User's Password

```javascript
use admin
db.changeUserPassword("adminUser", passwordPrompt())
```

Or using `updateUser`:

```javascript
use admin
db.updateUser("adminUser", {
  pwd: passwordPrompt()
})
```

## Listing and Dropping Users

List all users in the current database:

```javascript
db.getUsers()
```

Drop a user:

```javascript
db.dropUser("oldUser")
```

## Best Practices

- Never run MongoDB in production with authentication disabled.
- Use `passwordPrompt()` in mongosh instead of hardcoding passwords in commands.
- Create one database user per application and grant only the minimum required roles.
- Store application credentials in environment variables or a secrets manager, not in source code.
- Rotate passwords periodically and update connection strings accordingly.
- Use SCRAM-SHA-256 (the default for new users); avoid SCRAM-SHA-1 for new deployments.

## Summary

Enabling MongoDB authentication involves three steps: create an admin user while auth is disabled, enable `authorization: enabled` in `mongod.conf`, and restart MongoDB. After that, every connection must provide valid credentials. Always create application-specific users with minimal permissions rather than using the admin credentials in application code. Use `passwordPrompt()` in mongosh to avoid storing passwords in shell history.
