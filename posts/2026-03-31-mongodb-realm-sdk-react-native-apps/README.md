# How to Use Realm SDK with MongoDB for React Native Apps

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Realm, React Native, JavaScript, Mobile

Description: Learn how to use the Realm JavaScript SDK with MongoDB Atlas Device Sync in React Native apps to enable offline-first data storage and automatic cloud sync.

---

## Overview

The Realm JavaScript SDK integrates with React Native to provide an embedded local database backed by MongoDB Atlas. Your app reads and writes data locally at all times, and the SDK syncs changes to Atlas when the device is online.

## Installation

```bash
npm install realm @realm/react
npx pod-install  # iOS only
```

For Expo managed workflow, use the bare workflow or the Realm Expo plugin.

## Defining Your Object Schema

```javascript
import Realm from 'realm';

const TaskSchema = {
  name: 'Task',
  primaryKey: '_id',
  properties: {
    _id: 'objectId',
    name: 'string',
    isComplete: { type: 'bool', default: false },
    ownerId: 'string',
  },
};
```

## Connecting to Atlas App Services

```javascript
import Realm from 'realm';
import { AppProvider, UserProvider, RealmProvider } from '@realm/react';
```

Wrap your app with the required providers:

```javascript
export default function App() {
  return (
    <AppProvider id="YOUR-ATLAS-APP-ID">
      <UserProvider fallback={<LoginScreen />}>
        <RealmProvider
          schema={[TaskSchema]}
          sync={{
            flexible: true,
            initialSubscriptions: {
              update(subs, realm) {
                subs.add(realm.objects('Task'));
              },
            },
          }}
        >
          <TaskList />
        </RealmProvider>
      </UserProvider>
    </AppProvider>
  );
}
```

## Reading and Writing Data

Use the `@realm/react` hooks inside your components:

```javascript
import { useRealm, useQuery } from '@realm/react';
import Realm from 'realm';

function TaskList() {
  const realm = useRealm();
  const tasks = useQuery('Task', (collection) =>
    collection.filtered('isComplete == false')
  );

  const addTask = (name) => {
    realm.write(() => {
      realm.create('Task', {
        _id: new Realm.BSON.ObjectId(),
        name,
        ownerId: 'user-id',
        isComplete: false,
      });
    });
  };

  return (
    <>
      {tasks.map((task) => (
        <Text key={task._id.toString()}>{task.name}</Text>
      ))}
    </>
  );
}
```

## Handling Authentication

```javascript
import { useApp } from '@realm/react';

function LoginScreen() {
  const app = useApp();

  const login = async () => {
    await app.logIn(Realm.Credentials.emailPassword('user@example.com', 'password'));
  };

  return <Button title="Log In" onPress={login} />;
}
```

## Monitoring Sync Progress

```javascript
const realm = useRealm();

realm.syncSession?.addProgressNotification(
  Realm.ProgressDirection.Upload,
  Realm.ProgressMode.ReportIndefinitely,
  (transferred, total) => {
    console.log(`Upload progress: ${transferred}/${total}`);
  }
);
```

## Best Practices

- Scope subscriptions to the current user's data using `ownerId` filters to minimize data synced to the device.
- Handle `@realm/react` errors with the `onError` prop on `RealmProvider`.
- Avoid heavy computations inside `realm.write()` blocks to keep write transactions fast.

## Summary

The Realm JavaScript SDK with `@realm/react` hooks makes it simple to build offline-first React Native apps backed by MongoDB Atlas. Define your schema, wrap your app in providers, and use `useQuery` and `useRealm` hooks to interact with local data while sync runs transparently in the background.
