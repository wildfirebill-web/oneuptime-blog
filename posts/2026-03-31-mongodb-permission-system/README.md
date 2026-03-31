# How to Build a Permission System with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Permission, Role, Authorization, Security

Description: Learn how to design and implement a role-based permission system in MongoDB using roles, resource-level grants, and efficient query patterns for access control.

---

A well-designed permission system must answer two questions quickly: what can this user do, and can this user access this specific resource? MongoDB's document model supports both flat role assignments and fine-grained resource-level permissions in a single query.

## Modeling Roles and Users

Store roles in a dedicated collection. Each user document references their assigned roles.

```javascript
db.roles.insertMany([
  { _id: "admin", permissions: ["read", "write", "delete", "manage_users"] },
  { _id: "editor", permissions: ["read", "write"] },
  { _id: "viewer", permissions: ["read"] }
]);

db.users.insertOne({
  _id: "user-101",
  email: "alice@example.com",
  roles: ["editor"],
  createdAt: new Date()
});
```

## Modeling Resource-Level Permissions

Some resources require per-user overrides beyond what their role grants. Store explicit grants on the resource document.

```javascript
db.projects.insertOne({
  _id: "proj-42",
  name: "Website Redesign",
  owner: "user-001",
  members: [
    { userId: "user-101", permissions: ["read", "write"] },
    { userId: "user-202", permissions: ["read"] }
  ]
});
```

## Checking Global Role Permissions

Look up the user's roles and verify the required permission is present in any of them.

```javascript
async function hasGlobalPermission(userId, permission) {
  const user = await db.collection("users").findOne({ _id: userId });
  const roles = await db.collection("roles")
    .find({ _id: { $in: user.roles } })
    .toArray();
  return roles.some(r => r.permissions.includes(permission));
}
```

## Checking Resource-Level Access

Query the resource document directly, filtering on both the `_id` and the `members` array.

```javascript
async function canAccessProject(userId, projectId, permission) {
  const project = await db.collection("projects").findOne({
    _id: projectId,
    "members": {
      $elemMatch: {
        userId: userId,
        permissions: permission
      }
    }
  });
  return project !== null;
}
```

## Indexing for Access Control Queries

Index the `members.userId` field to avoid full collection scans when checking resource membership.

```javascript
db.projects.createIndex({ "members.userId": 1 });
db.users.createIndex({ roles: 1 });
```

## Listing Resources a User Can Access

Retrieve all projects where the user has at least read access.

```javascript
db.projects.find(
  {
    "members": {
      $elemMatch: { userId: "user-101", permissions: "read" }
    }
  },
  { name: 1, owner: 1 }
);
```

## Summary

A MongoDB permission system combines a roles collection for global capabilities and an embedded `members` array on each resource for fine-grained grants. Index `members.userId` to make resource-level access checks fast. The application layer merges role-level and resource-level checks to produce a final access decision without requiring complex SQL joins or a separate authorization service.
