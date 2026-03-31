# How to Implement Role-Based Access Control in Your App with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, RBAC, Authorization, Security, Application

Description: Learn how to implement role-based access control in your application using MongoDB, including storing roles, checking permissions, and enforcing access at the query level.

---

## Overview

Role-based access control (RBAC) restricts what users can do based on their assigned roles. MongoDB can store roles and permissions as part of your application's data model. This guide covers designing the schema, checking permissions in middleware, and scoping queries to enforce access control.

## Designing the RBAC Schema

Store roles and permissions in MongoDB collections:

```javascript
// roles collection
{
  _id: "admin",
  permissions: ["users:read", "users:write", "orders:read", "orders:write", "reports:read"]
}
{
  _id: "viewer",
  permissions: ["orders:read", "reports:read"]
}
{
  _id: "editor",
  permissions: ["orders:read", "orders:write"]
}

// users collection
{
  _id: ObjectId(),
  email: "alice@example.com",
  passwordHash: "...",
  roles: ["editor"],
  createdAt: new Date()
}
```

## Loading User Permissions

```javascript
async function getUserPermissions(userId, db) {
  const user = await db.collection("users").findOne(
    { _id: userId },
    { projection: { roles: 1 } }
  );
  if (!user) throw new Error("User not found");

  const roles = await db.collection("roles")
    .find({ _id: { $in: user.roles } })
    .toArray();

  const permissions = new Set();
  for (const role of roles) {
    for (const perm of role.permissions) {
      permissions.add(perm);
    }
  }
  return permissions;
}
```

## Express Middleware for Permission Checks

```javascript
function requirePermission(permission) {
  return async (req, res, next) => {
    try {
      const permissions = await getUserPermissions(req.user.id, req.app.locals.db);
      if (!permissions.has(permission)) {
        return res.status(403).json({ error: "Forbidden: insufficient permissions" });
      }
      req.userPermissions = permissions;
      next();
    } catch (err) {
      res.status(500).json({ error: "Permission check failed" });
    }
  };
}

// Usage
app.get("/api/orders", requirePermission("orders:read"), async (req, res) => {
  const orders = await db.collection("orders").find({}).toArray();
  res.json(orders);
});

app.post("/api/orders", requirePermission("orders:write"), async (req, res) => {
  const result = await db.collection("orders").insertOne(req.body);
  res.json(result);
});
```

## Scoping Queries by Ownership

For multi-tenant or row-level security, scope queries to the user's organization:

```javascript
async function getOrders(userId, orgId, db) {
  const permissions = await getUserPermissions(userId, db);

  const filter = permissions.has("orders:read:all")
    ? {}  // admins see all orders
    : { orgId: orgId };  // regular users see only their org's orders

  return db.collection("orders").find(filter).toArray();
}
```

## Caching Permissions

For high-traffic applications, cache permissions to avoid database lookups on every request:

```javascript
const cache = new Map();

async function getCachedPermissions(userId, db) {
  const cacheKey = userId.toString();
  if (cache.has(cacheKey)) return cache.get(cacheKey);

  const permissions = await getUserPermissions(userId, db);
  cache.set(cacheKey, permissions);
  setTimeout(() => cache.delete(cacheKey), 60000);  // expire after 60s
  return permissions;
}
```

## Summary

Implement RBAC in your MongoDB application by storing roles and permissions as collections, loading and caching user permissions at request time, and checking them in middleware before executing queries. Scope MongoDB queries by organization or ownership for row-level access control. Cache permissions with short TTLs to balance security freshness with performance.
