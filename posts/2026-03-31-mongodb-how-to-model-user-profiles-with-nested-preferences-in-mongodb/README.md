# How to Model User Profiles with Nested Preferences in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Data Modeling, User Profile, Preference, Schema Design

Description: Learn how to design a MongoDB schema for user profiles with nested preferences, notification settings, and extensible metadata using embedding and the attribute pattern.

---

## User Profile Data Model

User profiles typically combine core identity fields, preferences, settings, and extensible metadata. MongoDB's document model is ideal for this because preferences are naturally hierarchical and vary by application.

## Core User Profile Schema

```javascript
// users collection
{
  _id: ObjectId("u001"),

  // Identity
  email: "alice@example.com",
  username: "alice_smith",
  displayName: "Alice Smith",
  avatarUrl: "https://cdn.example.com/alice.jpg",
  bio: "Software engineer based in San Francisco",

  // Authentication
  passwordHash: "$2b$12$...",
  emailVerified: true,
  twoFactorEnabled: false,
  lastLoginAt: ISODate("2026-03-31T09:00:00Z"),

  // Nested preferences
  preferences: {
    theme: "dark",
    language: "en-US",
    timezone: "America/Los_Angeles",
    dateFormat: "MM/DD/YYYY",
    startOfWeek: "monday"
  },

  // Notification settings (nested object)
  notifications: {
    email: {
      enabled: true,
      marketing: false,
      productUpdates: true,
      weeklyDigest: true,
      securityAlerts: true
    },
    push: {
      enabled: true,
      mentions: true,
      directMessages: true,
      reminders: true
    },
    sms: {
      enabled: false,
      phone: null,
      securityAlerts: false
    }
  },

  // Privacy settings
  privacy: {
    profileVisibility: "public",         // public | friends | private
    showEmail: false,
    showLocation: true,
    allowMessagesFrom: "everyone"        // everyone | friends | none
  },

  // Account metadata
  role: "user",                          // user | admin | moderator
  plan: "pro",
  createdAt: ISODate("2025-01-15T00:00:00Z"),
  updatedAt: ISODate("2026-03-31T09:00:00Z")
}
```

## Reading User Preferences

Retrieve specific preference fields using projection:

```javascript
// Get display preferences only
db.users.findOne(
  { _id: ObjectId("u001") },
  { preferences: 1, displayName: 1, avatarUrl: 1 }
)

// Get notification settings
db.users.findOne(
  { email: "alice@example.com" },
  { notifications: 1 }
)
```

## Updating Nested Preferences

Use dot notation to update individual preference fields without overwriting the whole object:

```javascript
// Update a single preference
db.users.updateOne(
  { _id: ObjectId("u001") },
  {
    $set: {
      "preferences.theme": "light",
      "preferences.language": "en-GB",
      updatedAt: new Date()
    }
  }
)

// Disable all email notifications
db.users.updateOne(
  { _id: ObjectId("u001") },
  {
    $set: {
      "notifications.email.enabled": false,
      updatedAt: new Date()
    }
  }
)

// Enable push notifications and set mentions
db.users.updateOne(
  { _id: ObjectId("u001") },
  {
    $set: {
      "notifications.push.enabled": true,
      "notifications.push.mentions": true,
      updatedAt: new Date()
    }
  }
)
```

## Extensible Custom Preferences with Attribute Pattern

For application-specific or user-defined preferences that cannot be predicted ahead of time, use the attribute pattern:

```javascript
{
  _id: ObjectId("u001"),
  // ... core fields
  customSettings: [
    { k: "dashboard.defaultView", v: "kanban" },
    { k: "editor.fontSize", v: 14 },
    { k: "editor.tabSize", v: 2 },
    { k: "sidebar.collapsed", v: true },
    { k: "reports.defaultPeriod", v: "last_30_days" }
  ]
}
```

Get a specific custom setting:

```javascript
db.users.findOne(
  { _id: ObjectId("u001"), "customSettings.k": "dashboard.defaultView" },
  { "customSettings.$": 1 }
)
```

Set a custom setting:

```javascript
async function setCustomSetting(userId, key, value) {
  const result = await db.users.updateOne(
    { _id: userId, "customSettings.k": key },
    { $set: { "customSettings.$.v": value } }
  );

  if (result.matchedCount === 0) {
    // Setting doesn't exist yet, add it
    await db.users.updateOne(
      { _id: userId },
      { $push: { customSettings: { k: key, v: value } } }
    );
  }
}
```

## Social Connections

Store followers/following as arrays (for small counts) or in a separate collection (for large counts):

```javascript
// For < 1000 connections - embedded array
{
  _id: ObjectId("u001"),
  followingIds: [ObjectId("u002"), ObjectId("u003")],
  followerIds: [ObjectId("u004")],
  followerCount: 1,
  followingCount: 2
}

// For large social graphs - separate connections collection
{
  followerId: ObjectId("u001"),
  followedId: ObjectId("u002"),
  createdAt: ISODate("2026-01-20T00:00:00Z")
}
```

## Indexing User Profile Fields

```javascript
db.users.createIndex({ email: 1 }, { unique: true })
db.users.createIndex({ username: 1 }, { unique: true })
db.users.createIndex({ "preferences.timezone": 1 })
db.users.createIndex({ role: 1, plan: 1 })
db.users.createIndex({ createdAt: -1 })

// Custom settings lookup
db.users.createIndex({ "customSettings.k": 1 })
```

## Default Preferences Handling

Apply defaults when a user is created and when preferences are missing:

```javascript
const DEFAULT_PREFERENCES = {
  theme: "light",
  language: "en-US",
  timezone: "UTC",
  dateFormat: "YYYY-MM-DD",
  startOfWeek: "sunday"
};

const DEFAULT_NOTIFICATIONS = {
  email: { enabled: true, marketing: false, productUpdates: true, securityAlerts: true },
  push: { enabled: false, mentions: false, directMessages: false },
  sms: { enabled: false, phone: null }
};

function createUser(userData) {
  return db.users.insertOne({
    ...userData,
    preferences: { ...DEFAULT_PREFERENCES, ...(userData.preferences || {}) },
    notifications: DEFAULT_NOTIFICATIONS,
    privacy: { profileVisibility: "public", showEmail: false },
    createdAt: new Date(),
    updatedAt: new Date()
  });
}
```

## Summary

Model user profiles in MongoDB with a flat identity section and nested sub-documents for logically grouped settings like `preferences`, `notifications`, and `privacy`. Use dot-notation `$set` operations to update individual settings without overwriting sibling fields. For extensible or user-defined settings, apply the attribute pattern with a `customSettings` key-value array. Index unique fields like `email` and `username`, and use projections to fetch only the preference sections needed for each use case to minimize data transfer.
