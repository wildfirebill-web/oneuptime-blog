---
title: "How to Model User Profiles with Nested Preferences in MongoDB"
author: "nawazdhandala"
tags: ["MongoDB", "Schema design", "User", "Data modeling"]
description: "Learn how to model MongoDB user profiles with nested preference objects, handling sparse preferences, efficient updates, and queries across preference settings."
---

# How to Model User Profiles with Nested Preferences in MongoDB

User profiles often contain a mix of core identity fields and deeply nested preference settings. MongoDB's document model handles this naturally, but choosing the right embedding structure affects query performance, update simplicity, and schema flexibility.

## Core User Document

Start with a flat structure for core identity fields and embed preferences as a subdocument.

```javascript
db.users.insertOne({
  _id: ObjectId("64a1b2c3d4e5f6789abc0001"),
  username: "alice",
  email: "alice@example.com",
  displayName: "Alice Johnson",
  avatarUrl: "https://cdn.example.com/avatars/alice.jpg",
  role: "user",
  status: "active",
  createdAt: new Date("2024-01-15"),
  lastLoginAt: new Date("2024-06-15"),

  // Embedded preferences
  preferences: {
    language: "en-US",
    timezone: "America/Los_Angeles",
    theme: "dark",
    notifications: {
      email: {
        marketing: false,
        productUpdates: true,
        securityAlerts: true,
        weeklyDigest: false
      },
      push: {
        enabled: true,
        quietHours: { start: "22:00", end: "08:00" }
      },
      sms: {
        enabled: false,
        phoneNumber: null
      }
    },
    privacy: {
      profileVisible: true,
      showEmail: false,
      showActivity: true,
      allowTagging: true
    },
    accessibility: {
      fontSize: "medium",
      highContrast: false,
      reducedMotion: false,
      screenReader: false
    },
    dashboard: {
      defaultView: "grid",
      itemsPerPage: 25,
      pinnedWidgets: ["activity", "stats", "notifications"]
    }
  }
});
```

## Indexes for Preference-Based Queries

```javascript
// Find all users who opted into email notifications
db.users.createIndex({ "preferences.notifications.email.productUpdates": 1 });

// Find all users in a specific timezone
db.users.createIndex({ "preferences.timezone": 1 });

// Find users by language for localization batches
db.users.createIndex({ "preferences.language": 1 });
```

## Reading Preferences

```javascript
// Get a user's full preferences
const user = await db.collection("users").findOne(
  { _id: userId },
  { projection: { preferences: 1, displayName: 1, email: 1 } }
);
console.log(user.preferences.theme);
console.log(user.preferences.notifications.email.securityAlerts);
```

```javascript
// Get only notification preferences (partial projection)
const user = await db.collection("users").findOne(
  { _id: userId },
  { projection: { "preferences.notifications": 1 } }
);
```

## Updating Individual Preferences

Use dot notation to update individual nested fields without overwriting siblings.

```javascript
// Update a single preference
await db.collection("users").updateOne(
  { _id: userId },
  { $set: { "preferences.theme": "light" } }
);

// Update multiple nested preferences in one operation
await db.collection("users").updateOne(
  { _id: userId },
  {
    $set: {
      "preferences.language": "fr-FR",
      "preferences.timezone": "Europe/Paris",
      "preferences.notifications.email.weeklyDigest": true
    }
  }
);
```

## Handling Default Preferences

When a user first signs up, their preferences document may be sparse. Provide defaults either in the application layer or use `$setOnInsert` during registration.

```javascript
const DEFAULT_PREFERENCES = {
  language: "en-US",
  timezone: "UTC",
  theme: "system",
  notifications: {
    email: {
      marketing: false,
      productUpdates: true,
      securityAlerts: true,
      weeklyDigest: true
    },
    push: { enabled: true },
    sms: { enabled: false, phoneNumber: null }
  },
  privacy: {
    profileVisible: true,
    showEmail: false,
    showActivity: true,
    allowTagging: true
  }
};

// Merge stored preferences over defaults
function resolvePreferences(storedPreferences) {
  return {
    ...DEFAULT_PREFERENCES,
    ...storedPreferences,
    notifications: {
      ...DEFAULT_PREFERENCES.notifications,
      ...(storedPreferences.notifications || {}),
      email: {
        ...DEFAULT_PREFERENCES.notifications.email,
        ...(storedPreferences.notifications?.email || {})
      }
    }
  };
}
```

## Separating High-Churn Preferences

If certain preferences change very frequently (for example, last-viewed items or UI state), consider storing them in a separate collection to avoid excessive document churn on the main user document.

```javascript
// Separate collection for high-churn UI state
db.userSessionState.insertOne({
  _id: ObjectId(),
  userId: ObjectId("64a1b2c3d4e5f6789abc0001"),
  lastViewedItems: [
    ObjectId("64a1b2c3d4e5f6789abc3001"),
    ObjectId("64a1b2c3d4e5f6789abc3002")
  ],
  openTabs: ["dashboard", "settings"],
  sidebarCollapsed: false,
  updatedAt: new Date()
});

db.userSessionState.createIndex({ userId: 1 }, { unique: true });
```

## Preference Schema Versioning

As your application evolves, preferences gain new fields. Use a `prefVersion` field to apply migrations.

```javascript
async function getUserWithCurrentPreferences(db, userId) {
  const user = await db.collection("users").findOne({ _id: userId });
  if (!user) return null;

  // Apply preference migrations if needed
  if (!user.preferences.prefVersion || user.preferences.prefVersion < 2) {
    user.preferences = migratePreferencesToV2(user.preferences);
    await db.collection("users").updateOne(
      { _id: userId },
      { $set: { preferences: user.preferences } }
    );
  }

  return user;
}

function migratePreferencesToV2(prefs) {
  return {
    ...prefs,
    prefVersion: 2,
    accessibility: prefs.accessibility || {
      fontSize: "medium",
      highContrast: false,
      reducedMotion: false
    }
  };
}
```

## Aggregation: Find Users by Preference

```javascript
// Find all users who want security alert emails
const securityAlertUsers = await db.collection("users")
  .find({
    "preferences.notifications.email.securityAlerts": true,
    status: "active"
  })
  .project({ email: 1, displayName: 1 })
  .toArray();
```

## Summary

Model user profiles with a top-level `preferences` subdocument containing nested sections for notifications, privacy, accessibility, and UI settings. Use dot notation for targeted updates that change individual preference fields without overwriting the rest. Provide application-level defaults that merge over stored preferences so that new preference fields work correctly for existing users. For preference fields that change very frequently or need to be queried independently at scale, consider a separate collection. Add a `prefVersion` field to the preferences subdocument to support schema migrations as new preference types are added.
