# How to Define Data Access Rules in Atlas App Services

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, App Services, Security, Authorization

Description: Learn how to define fine-grained data access rules in MongoDB Atlas App Services to control which users can read or write specific documents and fields.

---

Atlas App Services uses a rule-based system to control data access. Rules evaluate on every read and write operation and can restrict access at the collection, document, or field level without any server-side code changes.

## Rule Types

Atlas App Services supports two layers of rules:

1. **Roles** - Define what a user can do (read, write, insert, delete)
2. **Filters** - Restrict which documents are included in query results

## Creating a Role in the Atlas UI

Navigate to **App Services > Rules**, select a collection, and click **Add Role**. Configure the role with:

```json
{
  "name": "owner",
  "apply_when": {
    "owner_id": "%%user.id"
  },
  "document_filters": {
    "read": { "owner_id": "%%user.id" },
    "write": { "owner_id": "%%user.id" }
  },
  "read": true,
  "write": true,
  "insert": true,
  "delete": true
}
```

The `apply_when` expression evaluates to true when the authenticated user's ID matches the document's `owner_id` field. This role is only applied to documents that the user owns.

## Defining Rules via Configuration File

For version-controlled deployments, define rules in JSON configuration files:

```json
{
  "database": "myApp",
  "collection": "tasks",
  "roles": [
    {
      "name": "owner",
      "apply_when": { "user_id": "%%user.id" },
      "read": true,
      "write": true,
      "insert": true,
      "delete": true,
      "search": true
    },
    {
      "name": "viewer",
      "apply_when": { "is_public": true },
      "read": true,
      "write": false,
      "insert": false,
      "delete": false
    }
  ]
}
```

Deploy this with the App Services CLI:

```bash
appservices push --remote="<App ID>"
```

## Field-Level Rules

Restrict access to specific fields using `fields`:

```json
{
  "name": "limited-viewer",
  "apply_when": {},
  "read": {
    "fields": {
      "title": { "read": true },
      "createdAt": { "read": true },
      "secretNotes": { "read": false }
    }
  }
}
```

Fields not listed default to the parent rule's permission. This lets you expose public fields while hiding sensitive data like internal notes or PII.

## Using Expansions in Rules

Atlas provides dynamic expansions to build context-aware rules:

```json
{
  "apply_when": {
    "department": "%%user.custom_data.department"
  }
}
```

This restricts access to documents where `department` matches the authenticated user's department stored in their custom data profile.

## Testing Rules

Use the Atlas App Services rule tester to verify rules before deploying:

```bash
# Simulate a request as a specific user
appservices accesslist create \
  --ip "0.0.0.0/0" \
  --comment "Allow all for testing"
```

You can also test rules programmatically using the Realm Web SDK:

```javascript
const app = new Realm.App({ id: "<App ID>" })
const user = await app.logIn(Realm.Credentials.emailPassword(email, password))
const collection = user.mongoClient("mongodb-atlas").db("myApp").collection("tasks")
const tasks = await collection.find({})
```

## Summary

Atlas App Services data access rules use roles with `apply_when` conditions to control who can read or write which documents. Combine document-level filtering, field-level restrictions, and user expansions to build fine-grained access control without managing a separate authorization layer. Store rules as JSON config files to keep them in version control and deploy them consistently across environments.
