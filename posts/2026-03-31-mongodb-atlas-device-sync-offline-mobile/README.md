# How to Use Atlas Device Sync for Offline-First Mobile Apps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Device Sync, Mobile, Offline

Description: Learn how to use MongoDB Atlas Device Sync to build offline-first mobile apps that automatically sync data between local Realm databases and MongoDB Atlas.

---

Atlas Device Sync (formerly Realm Sync) enables mobile apps to work fully offline and automatically sync changes to MongoDB Atlas when connectivity is restored. Data is stored locally in a Realm database and synced bidirectionally with the cloud.

## How Device Sync Works

Device Sync uses operational transforms to merge concurrent changes from multiple clients. Each device maintains a local Realm database. When online, changes are streamed to Atlas App Services, which resolves conflicts and propagates updates to all connected devices.

## Set Up Device Sync in the Atlas UI

1. Open your Atlas App Services application
2. Navigate to **Device Sync**
3. Choose **Flexible Sync** (recommended)
4. Select your linked MongoDB cluster and database
5. Define queryable fields (fields users can filter their sync subscription on)

```json
{
  "queryable_fields_names": ["owner_id", "status", "updatedAt"]
}
```

## Configure the Realm SDK (React Native)

Install the SDK:

```bash
npm install realm @realm/react
```

Define your schema:

```javascript
class Task extends Realm.Object {
  static schema = {
    name: "Task",
    primaryKey: "_id",
    properties: {
      _id: "objectId",
      owner_id: "string",
      title: "string",
      status: { type: "string", default: "open" },
      updatedAt: "date"
    }
  }
}
```

## Open a Synced Realm

```javascript
import Realm, { createRealmContext } from "@realm/react"

const { RealmProvider, useRealm, useQuery } = createRealmContext({
  schema: [Task],
  sync: {
    user: currentUser,
    flexible: true,
    onError: (session, error) => console.error(error)
  }
})
```

Add a subscription to define which data to sync:

```javascript
const realm = useRealm()

useEffect(() => {
  realm.subscriptions.update((subs) => {
    subs.add(realm.objects("Task").filtered("owner_id == $0", currentUser.id))
  })
}, [realm, currentUser])
```

## Write Locally, Sync Automatically

Writes to the local Realm are persisted immediately and synced in the background:

```javascript
realm.write(() => {
  realm.create("Task", {
    _id: new Realm.BSON.ObjectId(),
    owner_id: currentUser.id,
    title: "Buy groceries",
    status: "open",
    updatedAt: new Date()
  })
})
```

When offline, the change is queued locally. When connectivity is restored, Device Sync sends the change to Atlas automatically.

## Handle Conflict Resolution

Device Sync uses last-write-wins at the field level. If two clients update different fields of the same document while offline, both changes are merged. If two clients update the same field, the most recent timestamp wins.

For custom merge logic, use Atlas App Services Triggers to post-process synced documents:

```javascript
exports = function(changeEvent) {
  const doc = changeEvent.fullDocument
  // Custom merge or validation logic here
}
```

## Monitor Sync Health

Track sync session errors and latency in the Atlas App Services logs. Use operational monitoring (e.g., OneUptime) to alert on sync error spikes or increased Atlas trigger failure rates.

## Summary

Atlas Device Sync enables offline-first mobile apps by storing data locally in Realm and automatically syncing changes to MongoDB Atlas. Use Flexible Sync with subscriptions to control which data each user syncs. Writes are always local-first, ensuring the app works without a network connection, with automatic conflict resolution when clients reconnect.
