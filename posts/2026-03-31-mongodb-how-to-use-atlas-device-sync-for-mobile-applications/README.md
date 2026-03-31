# How to Use Atlas Device Sync for Mobile Applications

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, Device Sync, Mobile, Realm

Description: Set up MongoDB Atlas Device Sync to enable real-time data synchronization between mobile apps and MongoDB Atlas for offline-first mobile applications.

---

## What Is Atlas Device Sync?

Atlas Device Sync (formerly Realm Sync) enables bidirectional, real-time data synchronization between mobile or edge devices and MongoDB Atlas. Apps continue to work offline and sync automatically when connectivity is restored.

Key features:
- Offline-first architecture
- Automatic conflict resolution
- Fine-grained access control per user/role
- Available for iOS, Android, React Native, Flutter, and .NET MAUI

## Architecture Overview

```text
Mobile Device (Realm SDK)
        |
   Device Sync
        |
   Atlas App Services (sync logic, access rules)
        |
   MongoDB Atlas Cluster
```

Data flows through Atlas App Services, which enforces access rules before syncing to the Atlas cluster.

## Step 1: Create an App Services Application

1. In Atlas, click **App Services**
2. Click **Build your own App**
3. Name the app (e.g., `MyMobileApp`)
4. Link it to your Atlas cluster
5. Select a deployment region
6. Click **Create App Service**

## Step 2: Enable Device Sync

In the App Services UI:

1. Go to **Device Sync** in the sidebar
2. Click **Start Syncing**
3. Select **Flexible Sync** (recommended over Partition-based)
4. Select your cluster and database
5. Choose queryable fields (fields users can filter their sync subscription on)

Example queryable fields for a todo app:
```text
userId
status
priority
```

## Step 3: Define Data Access Rules

Access rules determine which documents a user can sync. In **Device Sync > Flexible Sync > Default Rules**:

```javascript
// Rule: users can only sync their own documents
{
  "%%user.id": "%%this.userId"
}
```

For more complex rules using roles:

```json
{
  "roles": [
    {
      "name": "owner",
      "apply_when": { "%%user.id": "%%this.userId" },
      "document_filters": {
        "read": true,
        "write": { "%%user.id": "%%this.userId" }
      },
      "read": true,
      "write": true
    }
  ]
}
```

## Step 4: Configure Authentication

Enable one or more authentication providers:

1. Go to **Authentication** in App Services sidebar
2. Enable **Email/Password** provider:

```javascript
// Configuration
{
  "autoConfirm": true,
  "resetPasswordUrl": "https://myapp.com/reset-password"
}
```

Or enable **Anonymous** for quick development:
- Toggle on **Allow users to log in anonymously**

## Step 5: Install the Realm SDK (React Native)

```bash
npm install realm @realm/react
```

## Step 6: Define Your Schema

```javascript
// models/Todo.js
import Realm from 'realm';

class Todo extends Realm.Object {
  static schema = {
    name: 'Todo',
    primaryKey: '_id',
    properties: {
      _id: 'objectId',
      userId: 'string',
      title: 'string',
      completed: { type: 'bool', default: false },
      priority: { type: 'int', default: 0 },
      createdAt: 'date'
    }
  };
}

export default Todo;
```

## Step 7: Connect and Sync in React Native

```javascript
import React from 'react';
import { AppProvider, UserProvider, RealmProvider, useRealm, useQuery } from '@realm/react';
import Realm from 'realm';
import Todo from './models/Todo';

const APP_ID = "myapp-abcde";

function TodoList() {
  const realm = useRealm();
  const todos = useQuery(Todo, (collection) =>
    collection.filtered("userId == $0", realm.currentUser.id).sorted("createdAt", true)
  );

  const addTodo = (title) => {
    realm.write(() => {
      realm.create(Todo, {
        _id: new Realm.BSON.ObjectId(),
        userId: realm.currentUser.id,
        title,
        completed: false,
        createdAt: new Date()
      });
    });
  };

  const toggleTodo = (todo) => {
    realm.write(() => {
      todo.completed = !todo.completed;
    });
  };

  return (
    // ...render todos
  );
}

export default function App() {
  return (
    <AppProvider id={APP_ID}>
      <UserProvider fallback={<LoginScreen />}>
        <RealmProvider
          schema={[Todo]}
          sync={{
            flexible: true,
            initialSubscriptions: {
              update(subs, realm) {
                subs.add(realm.objects(Todo).filtered(
                  "userId == $0", realm.currentUser.id
                ));
              }
            }
          }}
        >
          <TodoList />
        </RealmProvider>
      </UserProvider>
    </AppProvider>
  );
}
```

## Step 8: Handle Sync Errors

```javascript
<RealmProvider
  sync={{
    flexible: true,
    onError: (session, error) => {
      console.error("Sync error:", error.name, error.message);
      if (error.isFatal) {
        // Handle fatal errors - may need to re-open realm
        console.error("Fatal sync error - reconnecting");
      }
    }
  }}
>
```

## Step 9: Monitor Sync in Atlas

In Atlas App Services, go to **Device Sync > Logs** to view:
- Sync sessions per user
- Documents synced
- Error rates
- Bandwidth usage

## Summary

Atlas Device Sync enables offline-first mobile apps by syncing a local Realm database with MongoDB Atlas through App Services. Configure Flexible Sync with queryable fields and access rules to control per-user data access, install the Realm SDK for your mobile platform, define object schemas, and use sync providers to connect and subscribe to the documents your user needs. Data syncs automatically in the background and resolves conflicts using last-write-wins semantics.
