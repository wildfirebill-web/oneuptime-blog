# What Is MongoDB Atlas Device Sync

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Device Sync, Realm, Mobile Database, Offline-First

Description: MongoDB Atlas Device Sync synchronizes data between mobile or edge devices using Realm and MongoDB Atlas, enabling offline-first app development.

---

## What Is Atlas Device Sync

MongoDB Atlas Device Sync (formerly Realm Sync) is a feature that automatically synchronizes data between client-side Realm databases on mobile or edge devices and a MongoDB Atlas cluster in the cloud.

Device Sync is designed for offline-first applications - apps that continue to function when there is no network connection, storing data locally in Realm, and then automatically syncing changes to and from Atlas when connectivity is restored.

Atlas Device Sync handles conflict resolution, permission enforcement, and data routing automatically, reducing the complexity of building real-time, multi-device applications.

## Key Concepts

### Realm - The Local Database

Realm is an embedded, mobile-first database that runs on the device. It is not SQLite-based; it uses its own object-oriented storage engine optimized for mobile. The Realm SDK is available for:

- iOS (Swift/Objective-C)
- Android (Kotlin/Java)
- React Native (JavaScript/TypeScript)
- Flutter (Dart)
- .NET (C#)

### Atlas - The Cloud Backend

The MongoDB Atlas cluster acts as the source of truth in the cloud. Synced data from all devices is stored in Atlas collections, where it can be queried, aggregated, and integrated with other Atlas services.

### Sync Modes

Atlas Device Sync supports two sync modes:

- **Flexible Sync** - Clients subscribe to a subset of data based on query criteria. This is the recommended and more flexible mode.
- **Partition-Based Sync** - Older mode where data is partitioned by a partition key value. Flexible Sync is preferred for new projects.

## Setting Up Atlas Device Sync

### Step 1 - Enable Device Sync in Atlas

In the Atlas UI, navigate to your App Services application and enable Device Sync. Choose Flexible Sync and configure the queryable fields - these are the fields that clients can use in their sync subscriptions.

### Step 2 - Define a Schema

Define the data model. With Flexible Sync, you define a schema in Atlas App Services that maps to both the Realm object model on the client and the MongoDB collection on the server:

```json
{
  "title": "Task",
  "bsonType": "object",
  "required": ["_id", "name", "ownerId"],
  "properties": {
    "_id": { "bsonType": "objectId" },
    "name": { "bsonType": "string" },
    "isComplete": { "bsonType": "bool" },
    "ownerId": { "bsonType": "string" }
  }
}
```

### Step 3 - Configure Permissions

Set up permissions to control which users can read and write which data. A simple rule that allows users to access only their own data:

```json
{
  "roles": [
    {
      "name": "owner",
      "apply_when": { "ownerId": "%%user.id" },
      "document_filters": {
        "read": { "ownerId": "%%user.id" },
        "write": { "ownerId": "%%user.id" }
      },
      "read": true,
      "write": true
    }
  ]
}
```

### Step 4 - Connect from a Mobile Client

Here is an example using the Realm Swift SDK with Flexible Sync:

```swift
import RealmSwift

class Task: Object {
    @Persisted(primaryKey: true) var _id: ObjectId
    @Persisted var name: String = ""
    @Persisted var isComplete: Bool = false
    @Persisted var ownerId: String = ""
}

// Initialize the App
let app = App(id: "your-app-id")

// Authenticate
let user = try await app.login(credentials: .emailPassword(
    email: "user@example.com",
    password: "password"
))

// Open a synced Realm
var config = user.flexibleSyncConfiguration()
let realm = try await Realm(configuration: config)

// Subscribe to the user's tasks
try await realm.subscriptions.update {
    realm.subscriptions.append(
        QuerySubscription<Task>(name: "my-tasks") {
            $0.ownerId == user.id
        }
    )
}

// Create a task - it syncs to Atlas automatically
try realm.write {
    realm.add(Task(value: [
        "_id": ObjectId.generate(),
        "name": "Write unit tests",
        "ownerId": user.id
    ]))
}
```

### Step 5 - Kotlin Android Example

```kotlin
val app = App.create("your-app-id")

runBlocking {
    val user = app.login(Credentials.emailPassword("user@example.com", "password"))

    val config = SyncConfiguration.Builder(user, setOf(Task::class))
        .initialSubscriptions { realm ->
            add(
                realm.query<Task>("ownerId == $0", user.id),
                name = "my-tasks"
            )
        }
        .build()

    val realm = Realm.open(config)

    realm.write {
        copyToRealm(Task().apply {
            name = "Finish the report"
            ownerId = user.id
        })
    }
}
```

## Offline-First Behavior

When the device loses connectivity, the Realm SDK continues to work normally. All reads and writes operate against the local Realm database. Changes are queued and automatically synced when the device reconnects.

```swift
// This write works whether online or offline
try realm.write {
    let task = Task()
    task.name = "Offline task"
    task.ownerId = user.id
    realm.add(task)
}
// When connectivity is restored, this change is synced to Atlas
```

## Conflict Resolution

When multiple devices modify the same document offline and then sync, conflicts can arise. Atlas Device Sync uses operational transformation (OT) to merge changes automatically. The rules are:

- Field-level merges are preferred where possible
- Deletes take precedence over updates for the same document
- For conflicting field updates, the last write wins based on timestamp ordering

## Device Sync vs. Change Streams

| Feature | Device Sync | Change Streams |
|---|---|---|
| Target | Mobile / edge devices | Server-side consumers |
| Offline support | Yes | No |
| SDK required | Yes (Realm SDK) | No (any driver) |
| Conflict resolution | Automatic | Manual |
| Data direction | Bidirectional | Read-only stream |

## Summary

MongoDB Atlas Device Sync connects mobile and edge devices running Realm databases to a MongoDB Atlas cluster, enabling offline-first applications that sync automatically when connectivity is available. Flexible Sync allows clients to subscribe to specific subsets of data using query-based subscriptions, while Atlas App Services handles authentication, permissions, and conflict resolution in the cloud.
