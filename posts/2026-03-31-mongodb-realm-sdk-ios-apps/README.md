# How to Use Realm SDK with MongoDB for iOS Apps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Realm, iOS, Swift, Mobile

Description: Learn how to integrate the Realm Swift SDK with MongoDB Atlas to build offline-capable iOS apps with automatic data sync and local persistence.

---

## Overview

The Realm Swift SDK provides a local-first database for iOS apps with built-in sync to MongoDB Atlas. It lets you read and write data locally even when offline, then automatically syncs changes when connectivity is restored.

## Installing the Realm Swift SDK

Add the Realm Swift SDK via Swift Package Manager in Xcode:

1. Go to **File > Add Package Dependencies**
2. Enter: `https://github.com/realm/realm-swift`
3. Select the latest version and add both `Realm` and `RealmSwift` targets.

Or add it to your `Package.swift`:

```swift
dependencies: [
    .package(url: "https://github.com/realm/realm-swift", from: "10.0.0")
]
```

## Defining a Realm Object Model

```swift
import RealmSwift

class Task: Object {
    @Persisted(primaryKey: true) var _id: ObjectId
    @Persisted var name: String = ""
    @Persisted var isComplete: Bool = false
    @Persisted var ownerId: String = ""
}
```

All synced objects must have a primary key field named `_id`.

## Opening a Synced Realm

```swift
import RealmSwift

let app = App(id: "YOUR-ATLAS-APP-ID")

// Authenticate
let user = try await app.login(credentials: .emailPassword(email: "user@example.com", password: "password"))

// Configure sync
var config = user.flexibleSyncConfiguration()
let realm = try await Realm(configuration: config)

// Add a subscription
let subscriptions = realm.subscriptions
try await subscriptions.update {
    subscriptions.append(
        QuerySubscription<Task>(name: "userTasks") {
            $0.ownerId == user.id
        }
    )
}
```

## Writing Data Locally

```swift
try realm.write {
    let task = Task()
    task.name = "Buy groceries"
    task.ownerId = user.id
    realm.add(task)
}
```

Realm writes are synchronous and immediately available locally. The Realm SDK syncs them to Atlas in the background.

## Querying Data

```swift
let tasks = realm.objects(Task.self).where { $0.isComplete == false }
for task in tasks {
    print(task.name)
}
```

## Observing Changes with Combine

```swift
import Combine

var tokens: Set<AnyCancellable> = []

let tasks = realm.objects(Task.self)
tasks.collectionPublisher
    .sink(receiveCompletion: { _ in },
          receiveValue: { collection in
              print("Task count: \(collection.count)")
          })
    .store(in: &tokens)
```

## Handling Sync Errors

```swift
config.syncConfiguration?.errorHandler = { session, error in
    print("Sync error: \(error.localizedDescription)")
    // Log to your monitoring platform or retry
}
```

## Best Practices

- Always set `ownerId` on synced objects and use subscription queries filtered by `ownerId` to limit data downloaded to each device.
- Use `actor`-isolated realm instances in Swift concurrency to avoid threading issues.
- Monitor sync status in your Atlas App Services dashboard to catch write permission denials early.

## Summary

The Realm Swift SDK makes it straightforward to build offline-first iOS apps backed by MongoDB Atlas. Define your object model, open a flexible sync realm, and write/query data locally while the SDK handles background synchronization.
