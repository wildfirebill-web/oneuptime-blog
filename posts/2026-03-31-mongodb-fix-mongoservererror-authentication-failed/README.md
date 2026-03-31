# How to Fix MongoServerError: Authentication Failed in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Authentication, Error, Troubleshooting, Security

Description: Learn why MongoServerError Authentication Failed occurs in MongoDB and how to diagnose and fix credential, mechanism, and configuration issues step by step.

---

## Understanding the Error

`MongoServerError: Authentication failed` means the MongoDB server rejected the client's credentials. The error appears when the username, password, authentication database, or SASL mechanism does not match what the server expects.

A typical Node.js stack trace looks like:

```text
MongoServerError: Authentication failed.
    at Connection.onMessage (node_modules/mongodb/lib/cmap/connection.js)
```

## Common Causes and Fixes

### 1. Wrong Password or Username

The most frequent cause is a typo or stale password. Verify credentials in `mongosh`:

```bash
mongosh "mongodb://localhost:27017" -u admin -p --authenticationDatabase admin
```

Reset the password if needed:

```javascript
use admin
db.updateUser("myuser", { pwd: "newSecurePassword" })
```

### 2. Wrong Authentication Database

MongoDB stores user documents in a specific database (`authSource`). If your user was created in `admin` but you authenticate against `mydb`, the login fails.

Add `authSource` to the connection string:

```text
mongodb://myuser:mypassword@localhost:27017/mydb?authSource=admin
```

In Node.js with the native driver:

```javascript
const client = new MongoClient('mongodb://localhost:27017', {
  auth: { username: 'myuser', password: 'mypassword' },
  authSource: 'admin'
});
```

### 3. Wrong Authentication Mechanism

MongoDB supports SCRAM-SHA-256 (default since 4.0), SCRAM-SHA-1, and X.509. Some older clients default to SCRAM-SHA-1. Force the correct mechanism:

```javascript
const client = new MongoClient(uri, {
  authMechanism: 'SCRAM-SHA-256'
});
```

Or in the URI:

```text
mongodb://user:pass@host/db?authMechanism=SCRAM-SHA-256&authSource=admin
```

### 4. User Does Not Exist in That Database

Verify the user exists:

```javascript
use admin
db.getUsers()
```

Create the user if missing:

```javascript
db.createUser({
  user: "myuser",
  pwd: "securePassword",
  roles: [{ role: "readWrite", db: "mydb" }]
})
```

### 5. Special Characters in the Connection URI

If the password contains `@`, `/`, `:`, or `?`, they must be percent-encoded in the URI:

```python
from urllib.parse import quote_plus

password = quote_plus("p@ssw0rd!")
uri = f"mongodb://myuser:{password}@localhost:27017/mydb?authSource=admin"
```

### 6. Authentication Not Enabled on Server

If you just enabled `--auth` or `security.authorization: enabled` in `mongod.conf`, ensure at least one admin user exists before enforcing authentication, or you will lock yourself out.

```yaml
security:
  authorization: enabled
```

## Debugging Tips

Enable verbose logging on the client to see the SASL exchange:

```javascript
const client = new MongoClient(uri, { serverApi: { version: '1' }, monitorCommands: true });
client.on('commandStarted', (e) => console.log(JSON.stringify(e)));
```

Check the MongoDB server log for more detail:

```bash
sudo journalctl -u mongod --since "5 minutes ago" | grep -i "auth"
```

## Summary

`MongoServerError: Authentication Failed` is almost always caused by a credential mismatch - wrong password, wrong `authSource`, wrong mechanism, or URL-encoding issues. Verify each element of your connection string systematically, confirm the user exists in the correct database, and match the authentication mechanism to what your server version supports.
