# How to Use Atlas Triggers for Event-Driven Automation

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB Atlas, Triggers, Event-Driven, Serverless, Automation

Description: Learn how to use MongoDB Atlas Triggers to automate workflows on database changes, scheduled tasks, and authentication events using serverless functions.

---

## Introduction

MongoDB Atlas Triggers allow you to execute serverless JavaScript functions automatically in response to database change events, on a schedule, or during authentication events. They are built on Atlas App Services and enable event-driven architectures without managing server infrastructure. Common use cases include sending notifications, syncing data, auditing changes, and triggering downstream workflows.

## Types of Atlas Triggers

```text
Database Triggers  - Fire when documents are inserted, updated, deleted, or replaced
Scheduled Triggers - Fire on a cron schedule
Authentication Triggers - Fire on user login, creation, or deletion
```

## Creating a Database Trigger

In the Atlas UI, navigate to App Services - Triggers - Add Trigger and configure:

```text
Trigger Type: Database
Cluster: Your cluster name
Database: mydb
Collection: orders
Operation Type: Insert, Update, Delete (select as needed)
Full Document: Enabled (to access the full updated document)
```

Or define it via the Atlas App Services configuration JSON:

```json
{
  "name": "onOrderCreated",
  "type": "DATABASE",
  "config": {
    "service_name": "mongodb-atlas",
    "database": "mydb",
    "collection": "orders",
    "operation_types": ["INSERT"],
    "full_document": true,
    "full_document_before_change": false,
    "unordered": false
  },
  "function_name": "handleOrderCreated"
}
```

## Writing a Database Trigger Function

The trigger function receives a change event object:

```javascript
exports = async function (changeEvent) {
  const { operationType, fullDocument, documentKey } = changeEvent;

  if (operationType === "insert") {
    const order = fullDocument;

    // Send a notification (using a linked service)
    const emailService = context.services.get("my-email-service");
    await emailService.send({
      to: order.customerEmail,
      subject: `Order ${order.orderId} Confirmed`,
      body: `Your order for $${order.total} has been received.`
    });

    // Update a stats counter
    const mongodb = context.services.get("mongodb-atlas");
    const stats = mongodb.db("mydb").collection("orderStats");
    await stats.updateOne(
      { date: new Date().toISOString().split("T")[0] },
      { $inc: { count: 1, revenue: order.total } },
      { upsert: true }
    );
  }
};
```

## Trigger Function for Update Events

Handle document updates and audit changes:

```javascript
exports = async function (changeEvent) {
  const { operationType, fullDocument, updateDescription, documentKey } = changeEvent;

  if (operationType === "update") {
    const mongodb = context.services.get("mongodb-atlas");
    const audit = mongodb.db("mydb").collection("auditLog");

    await audit.insertOne({
      collection: "orders",
      documentId: documentKey._id,
      changedFields: Object.keys(updateDescription.updatedFields),
      updatedAt: new Date(),
      newValues: updateDescription.updatedFields
    });
  }
};
```

## Creating a Scheduled Trigger

Run a function every night at midnight UTC:

```json
{
  "name": "nightly-cleanup",
  "type": "SCHEDULED",
  "config": {
    "schedule": "0 0 * * *"
  },
  "function_name": "cleanupExpiredSessions"
}
```

The function:

```javascript
exports = async function () {
  const mongodb = context.services.get("mongodb-atlas");
  const sessions = mongodb.db("mydb").collection("sessions");

  const result = await sessions.deleteMany({
    expiresAt: { $lt: new Date() }
  });

  console.log(`Deleted ${result.deletedCount} expired sessions`);
};
```

## Authentication Triggers

Fire on user creation to initialize user data:

```json
{
  "name": "onUserCreated",
  "type": "AUTHENTICATION",
  "config": {
    "operation_type": "CREATE",
    "providers": ["local-userpass", "google", "facebook"]
  },
  "function_name": "initializeUserProfile"
}
```

```javascript
exports = async function ({ user }) {
  const mongodb = context.services.get("mongodb-atlas");
  const profiles = mongodb.db("mydb").collection("userProfiles");

  await profiles.insertOne({
    userId: user.id,
    email: user.data.email,
    createdAt: new Date(),
    plan: "free",
    preferences: {}
  });

  console.log(`Initialized profile for user ${user.id}`);
};
```

## Linking External Services

Atlas Triggers can call external HTTP endpoints:

```javascript
exports = async function (changeEvent) {
  const response = await context.http.post({
    url: "https://api.yourservice.com/webhook",
    headers: {
      "Content-Type": ["application/json"],
      "Authorization": [`Bearer ${context.values.get("webhook-secret")}`]
    },
    body: JSON.stringify({
      event: changeEvent.operationType,
      document: changeEvent.fullDocument
    })
  });

  if (response.statusCode !== 200) {
    throw new Error(`Webhook failed: ${response.statusCode}`);
  }
};
```

## Error Handling and Retries

Atlas Triggers automatically retry on error. Handle errors explicitly to prevent infinite retries:

```javascript
exports = async function (changeEvent) {
  try {
    await processEvent(changeEvent);
  } catch (err) {
    console.error(`Trigger error: ${err.message}`);

    // Log to error collection instead of retrying
    const mongodb = context.services.get("mongodb-atlas");
    await mongodb.db("mydb").collection("triggerErrors").insertOne({
      error: err.message,
      event: changeEvent.documentKey,
      timestamp: new Date()
    });

    // Do not re-throw - prevents retry for unrecoverable errors
  }
};
```

## Summary

Atlas Triggers provide a serverless, event-driven automation layer on top of MongoDB that eliminates the need for polling or change stream management in your application code. Use database triggers for real-time reactions to data changes like sending notifications or maintaining derived collections, scheduled triggers for periodic maintenance tasks, and authentication triggers for user lifecycle management. Store sensitive configuration in Atlas Values and Secrets rather than hardcoding them in function code.
