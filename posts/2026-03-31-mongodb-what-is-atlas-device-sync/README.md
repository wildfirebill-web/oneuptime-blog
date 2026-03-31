# What Is MongoDB Atlas Device Sync

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Device Sync, Mobile, Offline-First, Realm

Description: MongoDB Atlas Device Sync synchronizes data between Atlas and mobile or edge devices in real time, enabling offline-first applications with automatic conflict resolution.

---

## Deprecation Notice

> **Warning:** MongoDB Atlas Device Sync is deprecated as of September 2025. Check the [Atlas Device Sync end-of-life notice](https://www.mongodb.com/community/forums/t/update-to-end-of-life-and-deprecation-notice/297168) for the latest timeline and migration guidance.

## Overview

MongoDB Atlas Device Sync (formerly Realm Sync) is a data synchronization service that keeps data in sync between a MongoDB Atlas cluster and the Realm database running on mobile or edge devices. It enables offline-first application development: users can read and write data locally without an internet connection, and Device Sync automatically synchronizes changes with the backend when connectivity is restored, resolving conflicts along the way.

Device Sync is built into the Atlas App Services platform (formerly MongoDB Realm) and works with Realm SDKs for iOS (Swift), Android (Kotlin/Java), React Native, Flutter, and .NET (MAUI/Xamarin).

## Core Concepts

**Realm SDK** - A local embedded database that runs on the device. It provides reactive queries and object-oriented data modeling.

**Atlas App Services** - The cloud backend that hosts Device Sync, along with authentication, serverless functions, and triggers.

**Sync Protocol** - A bidirectional, operational-transform-based protocol that propagates changes between devices and the Atlas cluster.

**Flexible Sync** - The current sync mode, which lets each client define a query subscription to determine which documents to sync to the device.

## How Device Sync Works

1. The device opens a Realm with a sync configuration pointing to an Atlas App Services app.
2. The Realm SDK subscribes to a query (e.g., "all tasks belonging to the current user").
3. Only matching documents are synced down to the device.
4. Local writes are immediately persisted to the device Realm and then asynchronously synced to Atlas.
5. When changes arrive from Atlas or other devices, they are applied to the local Realm and trigger reactive UI updates.

## Setting Up Flexible Sync (Swift)

```swift
// Configure the sync
let appConfig = AppConfiguration(baseURL: "https://realm.mongodb.com")
let app = App(id: "your-app-id", configuration: appConfig)

// Log in
let user = try await app.login(credentials: .emailPassword(
    email: "user@example.com",
    password: "password"
))

// Open a synced Realm with a subscription
let config = user.flexibleSyncConfiguration()
let realm = try await Realm(configuration: config)

// Add a subscription to sync only the current user's tasks
let subscriptions = realm.subscriptions
try await subscriptions.update {
    subscriptions.append(QuerySubscription<Task>(name: "my-tasks") {
        $0.ownerId == user.id
    })
}
```

## Writing and Reading Data

```swift
// Write locally - syncs to Atlas automatically
try realm.write {
    let task = Task()
    task.title = "Buy groceries"
    task.ownerId = user.id
    realm.add(task)
}

// Query locally - reactive updates when sync delivers changes
let tasks = realm.objects(Task.self).where { $0.ownerId == user.id }
```

## Conflict Resolution

Device Sync uses operational transforms to merge concurrent edits. The resolution rules are:
- **Inserts** - Both inserts are applied; if the same `_id` is inserted by two clients, the last write wins on the field level
- **Updates to different fields** - Both applied (merge)
- **Updates to the same field** - Last write (by server timestamp) wins
- **Delete vs. update** - Delete wins

For custom conflict resolution, you can use Atlas triggers to apply business logic after sync.

## Permissions

Device Sync permissions are defined in App Services rules. You can control which users can read or write which documents using expressions based on user identity:

```json
{
  "rules": {
    "tasks": {
      "read": "%%user.id == ownerId",
      "write": "%%user.id == ownerId"
    }
  }
}
```

## When to Use Device Sync

- Mobile apps that need to function without internet connectivity
- Field service apps, point-of-sale, or logistics apps that operate in low-connectivity environments
- Real-time collaborative apps where changes must propagate to other users' devices
- IoT edge devices that need to sync sensor data back to a cloud backend

## Summary

MongoDB Atlas Device Sync enables offline-first mobile applications by synchronizing a local Realm database with MongoDB Atlas in real time. It handles connectivity interruptions, conflict resolution, and partial sync through query subscriptions, freeing developers from building complex sync infrastructure manually.
