# How to Create Users and Assign Roles in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, User Management, Role, Security, Authorization

Description: Create MongoDB users and assign built-in or custom roles to control what each application and operator can read, write, or administer.

---

## User Management Basics

MongoDB users are scoped to a database - the `authenticationDatabase`. Users authenticate against the database they were created in, typically `admin` for admin users and the application database for app users.

## Creating an Admin User

The admin user needs full access to manage the cluster:

```javascript
use admin

db.createUser({
  user: "clusterAdmin",
  pwd: passwordPrompt(),
  roles: [
    "userAdminAnyDatabase",
    "dbAdminAnyDatabase",
    "readWriteAnyDatabase",
    "clusterAdmin"
  ]
});
```

## Creating an Application Read/Write User

Application users should have the minimum required privileges:

```javascript
use myapp

db.createUser({
  user: "appUser",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
});
```

## Creating a Read-Only User

For analytics or reporting access:

```javascript
use myapp

db.createUser({
  user: "analyticsUser",
  pwd: passwordPrompt(),
  roles: [
    { role: "read", db: "myapp" }
  ]
});
```

## Assigning Roles Across Multiple Databases

A user can have roles in multiple databases:

```javascript
use admin

db.createUser({
  user: "multiDbUser",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "database1" },
    { role: "read", db: "database2" },
    { role: "dbAdmin", db: "database1" }
  ]
});
```

## Creating a Custom Role

When built-in roles are too broad, create a custom role with specific collection-level privileges:

```javascript
use myapp

db.createRole({
  role: "ordersReader",
  privileges: [
    {
      resource: { db: "myapp", collection: "orders" },
      actions: ["find"]
    },
    {
      resource: { db: "myapp", collection: "products" },
      actions: ["find"]
    }
  ],
  roles: []
});

db.createUser({
  user: "ordersDashboard",
  pwd: passwordPrompt(),
  roles: [{ role: "ordersReader", db: "myapp" }]
});
```

## Listing and Modifying Users

View all users in a database:

```javascript
use myapp
db.getUsers()
```

Update a user's password:

```javascript
db.changeUserPassword("appUser", passwordPrompt());
```

Grant additional roles:

```javascript
db.grantRolesToUser("appUser", [{ role: "dbAdmin", db: "myapp" }]);
```

Revoke roles:

```javascript
db.revokeRolesFromUser("appUser", [{ role: "dbAdmin", db: "myapp" }]);
```

## Summary

MongoDB users are created per database with `db.createUser()` and assigned one or more roles that define their access. Use built-in roles like `readWrite`, `read`, and `dbAdmin` for common use cases, and create custom roles with collection-level action granularity when finer control is needed. Always follow the principle of least privilege for application users.
