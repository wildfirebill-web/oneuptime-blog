# How to Use Built-In Roles in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Role, Security, Authorization, User Management

Description: A guide to MongoDB built-in roles including read, readWrite, dbAdmin, clusterAdmin, and when to use each for application and operator users.

---

## Categories of Built-In Roles

MongoDB built-in roles fall into four categories: database user roles, database administration roles, cluster administration roles, and all-database roles.

## Database User Roles

These are the most commonly used roles for application users.

**`read`** - Allows read operations on all non-system collections in a database:

```javascript
db.createUser({
  user: "reportUser",
  pwd: passwordPrompt(),
  roles: [{ role: "read", db: "analytics" }]
});
```

**`readWrite`** - Allows all `read` operations plus insert, update, delete, and index creation:

```javascript
db.createUser({
  user: "appUser",
  pwd: passwordPrompt(),
  roles: [{ role: "readWrite", db: "myapp" }]
});
```

## Database Administration Roles

**`dbAdmin`** - Allows schema management and statistics access but no read/write on user data:

```javascript
db.createUser({
  user: "schemaManager",
  pwd: passwordPrompt(),
  roles: [{ role: "dbAdmin", db: "myapp" }]
});
```

Typical actions: `createCollection`, `dropCollection`, `createIndex`, `dbStats`, `validate`.

**`userAdmin`** - Allows creation and modification of users and roles in a database:

```javascript
db.createUser({
  user: "dbUserAdmin",
  pwd: passwordPrompt(),
  roles: [{ role: "userAdmin", db: "myapp" }]
});
```

**`dbOwner`** - Combines `readWrite`, `dbAdmin`, and `userAdmin` - full control over one database:

```javascript
db.createUser({
  user: "dbOwner",
  pwd: passwordPrompt(),
  roles: [{ role: "dbOwner", db: "myapp" }]
});
```

## All-Database Roles

These roles apply across all databases and are assigned in the `admin` database:

```javascript
use admin

db.createUser({
  user: "globalAdmin",
  pwd: passwordPrompt(),
  roles: [
    "readAnyDatabase",       // read on all databases
    "readWriteAnyDatabase",  // readWrite on all databases
    "userAdminAnyDatabase",  // user management on all databases
    "dbAdminAnyDatabase"     // schema admin on all databases
  ]
});
```

## Cluster Administration Roles

**`clusterAdmin`** - Full cluster management including sharding and replication:

```javascript
use admin

db.createUser({
  user: "clusterMgr",
  pwd: passwordPrompt(),
  roles: ["clusterAdmin"]
});
```

**`clusterMonitor`** - Read-only access to cluster status (ideal for monitoring tools):

```javascript
db.createUser({
  user: "monitoringAgent",
  pwd: passwordPrompt(),
  roles: ["clusterMonitor"]
});
```

## Viewing What a Role Allows

To see the privileges granted by a built-in role:

```javascript
use admin
db.getRole("readWrite", { showPrivileges: true })
```

## Choosing the Right Role

| User Type | Recommended Role |
|-----------|-----------------|
| Application backend | `readWrite` on app DB |
| Reporting service | `read` on app DB |
| Schema migrations | `dbAdmin` + `readWrite` |
| Monitoring agent | `clusterMonitor` |
| DBA full access | `dbOwner` per DB |

## Summary

MongoDB built-in roles cover the most common access patterns without requiring custom role creation. Use `readWrite` for application users, `read` for reporting, `dbAdmin` for schema management, and `clusterAdmin` sparingly for cluster administrators. Always scope roles to the specific database to follow the principle of least privilege.
