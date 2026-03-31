# How to Use mongosh to Manage Users and Roles in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongosh, User, Role

Description: Learn how to create, modify, and delete MongoDB users and roles using mongosh commands for database access control and least-privilege security.

---

## Prerequisites

Connect to MongoDB as a user with the `userAdminAnyDatabase` role:

```bash
mongosh "mongodb://admin:password@localhost:27017/admin"
```

## Creating Users

Create a user in a specific database:

```javascript
use mydb
db.createUser({
  user: "appUser",
  pwd: "SecurePass123!",
  roles: [
    { role: "readWrite", db: "mydb" }
  ]
});
```

Create a read-only user:

```javascript
db.createUser({
  user: "reportUser",
  pwd: "ReportPass456!",
  roles: [
    { role: "read", db: "mydb" },
    { role: "read", db: "analytics" }
  ]
});
```

## Listing Users

```javascript
// List users in current database
db.getUsers();

// Get a specific user
db.getUser("appUser");

// List all users across all databases
use admin
db.system.users.find().pretty();
```

## Updating a User's Password

```javascript
db.updateUser("appUser", {
  pwd: "NewSecurePass789!"
});
```

## Adding Roles to an Existing User

```javascript
db.grantRolesToUser("appUser", [
  { role: "readWrite", db: "analytics" }
]);
```

## Revoking Roles from a User

```javascript
db.revokeRolesFromUser("appUser", [
  { role: "readWrite", db: "analytics" }
]);
```

## Deleting a User

```javascript
db.dropUser("reportUser");
```

## Creating Custom Roles

Define a role with specific privileges:

```javascript
use admin
db.createRole({
  role: "orderManager",
  privileges: [
    {
      resource: { db: "mydb", collection: "orders" },
      actions: ["find", "insert", "update"]
    },
    {
      resource: { db: "mydb", collection: "products" },
      actions: ["find"]
    }
  ],
  roles: []
});
```

Assign the custom role to a user:

```javascript
db.grantRolesToUser("appUser", [{ role: "orderManager", db: "admin" }]);
```

## Listing Roles

```javascript
// Built-in and custom roles in current database
db.getRoles({ showBuiltinRoles: true });

// Get details of a specific role
db.getRole("orderManager", { showPrivileges: true });
```

## Modifying Custom Roles

```javascript
// Add privileges to a role
db.grantPrivilegesToRole("orderManager", [
  {
    resource: { db: "mydb", collection: "customers" },
    actions: ["find"]
  }
]);

// Remove privileges from a role
db.revokePrivilegesFromRole("orderManager", [
  {
    resource: { db: "mydb", collection: "products" },
    actions: ["find"]
  }
]);
```

## Dropping a Custom Role

```javascript
db.dropRole("orderManager");
```

## Summary

Use mongosh's `db.createUser()`, `db.updateUser()`, and `db.dropUser()` to manage MongoDB users. Grant and revoke roles with `db.grantRolesToUser()` and `db.revokeRolesFromUser()`. For fine-grained control, create custom roles with `db.createRole()` specifying exact resource and action combinations. Always follow the least-privilege principle by granting only the permissions each user requires.
