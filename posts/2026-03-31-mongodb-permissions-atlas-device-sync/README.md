# How to Configure Permissions for Atlas Device Sync

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Device Sync, Permission, Security

Description: Learn how to define roles and rules for Atlas Device Sync to control which users can read and write synced data based on ownership and custom logic.

---

## Overview

Atlas Device Sync enforces data access using server-side permissions defined in Atlas App Services. These permissions determine which documents a user can sync to their device and whether they can insert, update, or delete records.

## Permission Models

Atlas Device Sync supports two permission approaches:

- **Default roles** - Simple, globally applied rules using predefined templates.
- **Collection-level rules** - Fine-grained per-collection rules with custom expressions.

## Using the "Users can only read and write their own data" Template

In the Atlas App Services UI:

1. Navigate to **App Services > Your App > Device Sync**.
2. Under **Permissions**, select **Users can only read and write their own data**.
3. Set the **User ID Field** to `ownerId`.

This generates a rule that restricts each user to documents where `ownerId` equals their user ID.

## Defining Custom Collection Rules

For more complex scenarios, define rules using JSON expressions:

```json
{
  "roles": [
    {
      "name": "owner",
      "apply_when": { "ownerId": "%%user.id" },
      "read": true,
      "write": true,
      "insert": true,
      "delete": true
    },
    {
      "name": "viewer",
      "apply_when": { "teamId": "%%user.custom_data.teamId" },
      "read": true,
      "write": false,
      "insert": false,
      "delete": false
    }
  ]
}
```

This configuration allows owners to fully manage their documents and team members to read shared documents.

## Populating Custom User Data for Rule Expressions

Custom user data is stored in a MongoDB collection and linked to the user. Create a `users` collection:

```javascript
db.users.insertOne({
  _id: "user-id-123",
  teamId: "team-abc",
  role: "admin"
})
```

Configure the custom user data source in **App Services > App Users > Custom User Data** to point to this collection with `_id` as the user ID field.

## Validating Permissions with Function Rules

For complex authorization logic, use a Function rule:

```javascript
exports = async function(user, document) {
  // Allow write only if user is in the admin list
  const adminCollection = context.services.get("mongodb-atlas")
    .db("mydb").collection("admins");
  const admin = await adminCollection.findOne({ userId: user.id });
  return admin !== null;
};
```

## Testing Permissions

Use the **Rule Tester** in the Atlas App Services UI to simulate a user's access to a specific document before deploying. This prevents accidental data exposure.

## Monitoring Permission Violations

Permission denials appear as **compensating writes** in the Atlas App Services logs. Check them via:

```bash
curl -u "PUBLIC_KEY:PRIVATE_KEY" --digest \
  "https://realm.mongodb.com/api/admin/v3.0/groups/{groupId}/apps/{appId}/logs?type=sync&start=2026-01-01T00:00:00Z"
```

## Best Practices

- Always include `ownerId` in synced documents and use it as the primary permission filter.
- Use custom user data roles to implement team-based or organization-based data sharing.
- Regularly review App Services logs for compensating write events indicating permission mismatches on the client.
- Apply least-privilege rules - grant write access only to collections that require it.

## Summary

Atlas Device Sync permissions are enforced server-side through roles and rule expressions configured in App Services. Start with the built-in ownership template and extend with custom roles, custom user data, and Function rules to implement fine-grained multi-tenant access control.
