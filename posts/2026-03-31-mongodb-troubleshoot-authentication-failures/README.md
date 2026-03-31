# How to Troubleshoot MongoDB Authentication Failures

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Authentication, Security, Troubleshooting, SCRAM

Description: Diagnose and fix MongoDB authentication failures including wrong passwords, missing users, incorrect auth databases, TLS issues, and SCRAM mechanism mismatches.

---

## Overview

MongoDB authentication failures surface as "Authentication failed", "not authorized", or "bad auth" errors. The cause is almost always one of: wrong credentials, wrong authentication database, TLS certificate mismatch, or a driver/server SCRAM mechanism incompatibility. This guide walks through each scenario.

## Step 1: Identify the Error Message

```text
Common MongoDB authentication errors:

"Authentication failed."
  - Wrong username or password
  - Wrong authenticationDatabase

"not authorized on <db> to execute command"
  - User exists but lacks the required role for this database

"bad auth : no sasl support in this build"
  - MongoDB was compiled without SASL support (uncommon)

"SCRAM authentication exchange failed"
  - Mechanism mismatch between client and server

"SSL/TLS connection error"
  - Certificate validation failure on x.509 auth
```

## Step 2: Verify the User Exists

```javascript
// Connect without auth (only possible if auth is not yet enabled)
// or use a different admin account
use admin
db.getUser('myapp')

// List all users in a database
db.getUsers()

// If user does not exist, create it
db.createUser({
  user: 'myapp',
  pwd: 'S3cureP@ssw0rd',
  roles: [{ role: 'readWrite', db: 'appdb' }]
})
```

## Step 3: Check the Authentication Database

MongoDB users are created in a specific database. The `authenticationDatabase` in the connection string must match where the user was created.

```bash
# Wrong - user was created in admin but connecting to appdb
mongosh "mongodb://myapp:password@host/appdb"

# Correct - specify the authSource
mongosh "mongodb://myapp:password@host/appdb?authSource=admin"
```

In the Node.js driver:

```javascript
const client = new MongoClient('mongodb://localhost:27017', {
  auth: {
    username: 'myapp',
    password: 'S3cureP@ssw0rd'
  },
  authSource: 'admin'   // This must match where the user was created
});
```

## Step 4: Reset a User Password

```javascript
use admin
db.changeUserPassword('myapp', 'NewStr0ngP@ssword!')

// Or use the updateUser command for more control
db.updateUser('myapp', {
  pwd: 'NewStr0ngP@ssword!',
  passwordDigestor: 'server'   // or 'client' for older drivers
})
```

## Step 5: Fix SCRAM Mechanism Mismatches

MongoDB 4.0+ supports both SCRAM-SHA-1 and SCRAM-SHA-256. Some older drivers default to SCRAM-SHA-1.

```javascript
// Check which mechanisms a user supports
use admin
db.getUser('myapp').mechanisms
// ["SCRAM-SHA-1", "SCRAM-SHA-256"]

// Force a specific mechanism in the connection string
```

```bash
mongosh "mongodb://myapp:password@host/admin?authMechanism=SCRAM-SHA-256"
```

In the Node.js driver:

```javascript
const client = new MongoClient(uri, {
  authMechanism: 'SCRAM-SHA-256'
});
```

## Step 6: Diagnose "not authorized" Role Errors

```javascript
// Check what roles a user has
use admin
db.getUser('myapp').roles

// Grant missing roles
db.grantRolesToUser('myapp', [
  { role: 'readWrite', db: 'appdb' },
  { role: 'read', db: 'reporting' }
])

// Revoke excess roles
db.revokeRolesFromUser('myapp', [
  { role: 'dbAdmin', db: 'appdb' }
])
```

## Step 7: Enable Auth Logging for Diagnosis

```bash
# Set log verbosity for access control
mongosh --eval 'db.adminCommand({ setParameter: 1, logLevel: 1 })'

# Watch the log for auth events
tail -f /var/log/mongodb/mongod.log | grep -i "auth\|SCRAM\|access"
```

## Step 8: Verify Auth is Enabled on the Server

```javascript
// Check if auth is enabled
db.adminCommand({ getCmdLineOpts: 1 }).parsed.security
// Should return { authorization: 'enabled' }
```

If auth is not enabled, add it to `mongod.conf`.

```yaml
security:
  authorization: enabled
```

## Summary

MongoDB authentication failures are caused by wrong credentials, wrong `authenticationDatabase`, missing roles, or SCRAM mechanism mismatches. Always specify `authSource` in connection strings to match where the user was created, use `db.getUser()` to verify user existence and roles, enable auth logging temporarily to capture detailed error context, and ensure the driver's `authMechanism` is compatible with the server's supported mechanisms.
