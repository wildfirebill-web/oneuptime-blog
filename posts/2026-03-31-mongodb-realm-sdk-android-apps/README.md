# How to Use Realm SDK with MongoDB for Android Apps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Realm, Android, Kotlin, Mobile

Description: Learn how to integrate the Realm Kotlin SDK with MongoDB Atlas to build offline-capable Android apps with automatic data sync and local Realm persistence.

---

## Overview

The Realm Kotlin SDK provides a local-first embedded database for Android apps that syncs with MongoDB Atlas Device Sync. It supports offline reads and writes, with automatic conflict resolution when connectivity is restored.

## Adding the Realm Kotlin SDK

In your project-level `build.gradle.kts`:

```kotlin
plugins {
    id("io.realm.kotlin") version "1.13.0" apply false
}
```

In your app-level `build.gradle.kts`:

```kotlin
plugins {
    id("io.realm.kotlin")
}

dependencies {
    implementation("io.realm.kotlin:library-sync:1.13.0")
}
```

## Defining a Realm Object Model

```kotlin
import io.realm.kotlin.types.RealmObject
import io.realm.kotlin.types.annotations.PrimaryKey
import org.mongodb.kbson.ObjectId

class Task : RealmObject {
    @PrimaryKey
    var _id: ObjectId = ObjectId()
    var name: String = ""
    var isComplete: Boolean = false
    var ownerId: String = ""
}
```

## Initializing the App and Authenticating

```kotlin
import io.realm.kotlin.mongodb.App
import io.realm.kotlin.mongodb.Credentials

val app = App.create("YOUR-ATLAS-APP-ID")
val user = app.login(Credentials.emailPassword("user@example.com", "password"))
```

## Opening a Flexible Sync Realm

```kotlin
import io.realm.kotlin.mongodb.sync.SyncConfiguration

val config = SyncConfiguration.Builder(user, setOf(Task::class))
    .initialSubscriptions { realm ->
        add(realm.query<Task>("ownerId == $0", user.id), name = "userTasks")
    }
    .build()

val realm = Realm.open(config)
```

## Writing and Querying Data

```kotlin
// Write
realm.write {
    copyToRealm(Task().apply {
        name = "Complete Android tutorial"
        ownerId = user.id
    })
}

// Query
val incompleteTasks = realm.query<Task>("isComplete == false").find()
incompleteTasks.forEach { println(it.name) }
```

## Observing Changes with Kotlin Flow

```kotlin
import kotlinx.coroutines.flow.collect

realm.query<Task>().asFlow().collect { changes ->
    println("Updated task count: ${changes.list.size}")
}
```

## Handling Sync Errors

```kotlin
val config = SyncConfiguration.Builder(user, setOf(Task::class))
    .errorHandler { session, error ->
        println("Sync error: ${error.message}")
    }
    .build()
```

## Best Practices

- Always filter subscriptions by `ownerId` to avoid downloading data that does not belong to the current user.
- Run realm queries and writes inside coroutines using `withContext(Dispatchers.IO)` to keep the main thread responsive.
- Use `realm.close()` in your lifecycle `onDestroy` to release resources and prevent memory leaks.
- Enable Atlas Device Sync rules in App Services to enforce server-side write permission checks.

## Summary

The Realm Kotlin SDK makes it easy to add offline-first capabilities to Android apps with automatic sync to MongoDB Atlas. Define your data model, configure flexible sync subscriptions, and use Kotlin coroutines and Flow to reactively observe data changes while the SDK handles all synchronization in the background.
