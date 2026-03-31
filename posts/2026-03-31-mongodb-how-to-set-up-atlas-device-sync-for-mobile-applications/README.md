# How to Set Up Atlas Device Sync for Mobile Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Device Sync, Mobile, Realm, Offline-First

Description: Learn how to configure MongoDB Atlas Device Sync to enable real-time data synchronization between mobile apps and your Atlas cluster.

---

## What Is Atlas Device Sync?

Atlas Device Sync (formerly Realm Sync) enables real-time, bidirectional data synchronization between mobile and edge devices running the Realm SDK and your MongoDB Atlas cluster. It supports offline-first applications - devices work locally when offline and automatically sync when connectivity is restored.

Key features:
- Offline-first operation
- Automatic conflict resolution
- Fine-grained permission controls
- Client-side encryption
- Available for iOS, Android, React Native, Flutter, and .NET

## Prerequisites

```text
- MongoDB Atlas account with an M0 or higher cluster
- Atlas App Services application configured
- Realm SDK in your mobile project
```

## Enabling Device Sync in Atlas App Services

1. Navigate to Atlas App Services at services.cloud.mongodb.com
2. Create or select an App
3. Click **Device Sync** in the left sidebar
4. Click **Enable Sync**
5. Choose **Flexible Sync** (recommended over Partition-Based Sync)
6. Select your linked Atlas cluster
7. Choose a **Database** to sync data from
8. Configure queryable fields (fields users can use to filter which data syncs)

## Configuring Flexible Sync

Flexible Sync lets clients subscribe to subsets of data using queries:

```javascript
// Atlas App Services - configure queryable fields
// In App Services UI: Device Sync > Queryable Fields
// Add fields that clients need to filter on:
// - userId (for user-specific data)
// - teamId (for team-based data)
// - isPublic (for public/private data)
```

## Setting Up Permissions

Configure which users can read or write which documents:

```javascript
// App Services > Rules > Default Rule
// Apply to collection: tasks
{
  "roles": [
    {
      "name": "owner",
      "apply_when": { "userId": "%%user.id" },
      "document_filters": {
        "read": true,
        "write": true
      },
      "read": true,
      "write": true
    },
    {
      "name": "viewer",
      "apply_when": {},
      "document_filters": {
        "read": { "isPublic": true }
      },
      "read": true,
      "write": false
    }
  ]
}
```

## iOS Swift Integration

Install the Realm SDK via Swift Package Manager, then configure sync:

```swift
import RealmSwift

// Define a synced object model
class Task: Object {
    @Persisted(primaryKey: true) var _id: ObjectId
    @Persisted var title: String
    @Persisted var isCompleted: Bool = false
    @Persisted var userId: String  // queryable field
}

// Authenticate user
let app = App(id: "your-app-id")

func setupSync() async throws {
    let user = try await app.login(credentials: .emailPassword(
        email: "user@example.com",
        password: "password123"
    ))

    // Open a synced Realm
    var config = user.flexibleSyncConfiguration()
    let realm = try await Realm(configuration: config)

    // Subscribe to data for this user
    let subscriptions = realm.subscriptions
    try await subscriptions.update {
        subscriptions.append(
            QuerySubscription<Task>(name: "my-tasks") {
                $0.userId == user.id
            }
        )
    }

    // Access synced data - works offline too
    let tasks = realm.objects(Task.self)
    print("Tasks: \(tasks.count)")
}
```

## Android Kotlin Integration

```kotlin
import io.realm.kotlin.Realm
import io.realm.kotlin.ext.query
import io.realm.kotlin.mongodb.App
import io.realm.kotlin.mongodb.Credentials
import io.realm.kotlin.mongodb.sync.SyncConfiguration

// Authenticate and configure sync
val app = App.create("your-app-id")
val user = app.login(Credentials.emailPassword("user@example.com", "pass"))

val config = SyncConfiguration.Builder(user, setOf(Task::class))
    .initialSubscriptions { realm ->
        add(
            realm.query<Task>("userId == $0", user.id),
            name = "my-tasks"
        )
    }
    .build()

val realm = Realm.open(config)

// Write locally - syncs automatically
realm.write {
    copyToRealm(Task().apply {
        title = "Buy groceries"
        userId = user.id
    })
}
```

## React Native Integration

```javascript
import Realm, { App, Credentials } from "realm"

const app = new App({ id: "your-app-id" })

// Define schema
const TaskSchema = {
  name: "Task",
  primaryKey: "_id",
  properties: {
    _id: "objectId",
    title: "string",
    isCompleted: { type: "bool", default: false },
    userId: "string"
  }
}

async function setupSync() {
  const user = await app.logIn(
    Credentials.emailPassword("user@example.com", "password123")
  )

  const config = {
    schema: [TaskSchema],
    sync: {
      user,
      flexible: true,
      initialSubscriptions: {
        update(subs, realm) {
          subs.add(realm.objects("Task").filtered("userId == $0", user.id))
        }
      }
    }
  }

  const realm = await Realm.open(config)
  return realm
}
```

## Monitoring Sync Status

```javascript
// React Native: listen for sync errors
const config = {
  sync: {
    user,
    flexible: true,
    onError: (session, error) => {
      console.error("Sync error:", error.message)
      if (error.name === "ClientResetError") {
        // Handle client reset
      }
    }
  }
}

// Check connection state
const realm = await Realm.open(config)
console.log("Sync state:", realm.syncSession.state)
// "active" | "inactive" | "waiting_for_access_token"
```

## Summary

Atlas Device Sync connects mobile apps to MongoDB Atlas using the Realm SDK, enabling offline-first data access with automatic synchronization. Set up Flexible Sync in Atlas App Services, configure queryable fields and permissions, then integrate the Realm SDK in iOS, Android, or React Native to open synced realms with user-specific subscriptions. Data written locally syncs automatically when connectivity is available.
