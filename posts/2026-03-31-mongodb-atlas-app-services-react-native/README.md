# How to Use Atlas App Services with React Native

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas, React Native, Mobile, App Service

Description: Learn how to integrate MongoDB Atlas App Services into a React Native app using the Realm SDK for authentication, data access, and offline-first sync.

---

MongoDB Atlas App Services provides a complete backend for React Native apps - authentication, data access rules, serverless functions, and offline sync via Device Sync. The `@realm/react` package makes integration straightforward.

## Installation

```bash
npm install realm @realm/react
cd ios && pod install
```

## Initialize the App

Create an App Services instance at the root of your app:

```javascript
import { AppProvider, UserProvider, RealmProvider } from "@realm/react"
import Realm from "realm"

export default function App() {
  return (
    <AppProvider id="<YOUR_APP_ID>">
      <UserProvider fallback={<LoginScreen />}>
        <MainApp />
      </UserProvider>
    </AppProvider>
  )
}
```

`AppProvider` initializes the connection to your Atlas App Services application. `UserProvider` ensures a user is authenticated before rendering its children.

## Authentication

Build a login screen using the `useApp` hook:

```javascript
import { useApp } from "@realm/react"

function LoginScreen() {
  const app = useApp()
  const [email, setEmail] = React.useState("")
  const [password, setPassword] = React.useState("")

  const handleLogin = async () => {
    try {
      const credentials = Realm.Credentials.emailPassword(email, password)
      await app.logIn(credentials)
    } catch (err) {
      Alert.alert("Login failed", err.message)
    }
  }

  return (
    <View>
      <TextInput value={email} onChangeText={setEmail} placeholder="Email" />
      <TextInput value={password} onChangeText={setPassword} secureTextEntry />
      <Button title="Log In" onPress={handleLogin} />
    </View>
  )
}
```

## Define a Schema and Open a Realm

```javascript
class Task extends Realm.Object {
  static schema = {
    name: "Task",
    primaryKey: "_id",
    properties: {
      _id: "objectId",
      owner_id: "string",
      title: "string",
      isComplete: { type: "bool", default: false }
    }
  }
}
```

Wrap your main content with `RealmProvider` for Device Sync:

```javascript
import { useUser } from "@realm/react"

function MainApp() {
  const user = useUser()

  return (
    <RealmProvider
      schema={[Task]}
      sync={{
        user,
        flexible: true,
        initialSubscriptions: {
          update(subs, realm) {
            subs.add(
              realm.objects("Task").filtered("owner_id == $0", user.id)
            )
          }
        }
      }}
    >
      <TaskList />
    </RealmProvider>
  )
}
```

## Read and Write Data

```javascript
import { useRealm, useQuery } from "@realm/react"

function TaskList() {
  const realm = useRealm()
  const tasks = useQuery("Task")
  const user = useUser()

  const addTask = (title) => {
    realm.write(() => {
      realm.create("Task", {
        _id: new Realm.BSON.ObjectId(),
        owner_id: user.id,
        title,
        isComplete: false
      })
    })
  }

  const toggleTask = (task) => {
    realm.write(() => {
      task.isComplete = !task.isComplete
    })
  }

  return (
    <FlatList
      data={tasks}
      keyExtractor={(item) => item._id.toString()}
      renderItem={({ item }) => (
        <TouchableOpacity onPress={() => toggleTask(item)}>
          <Text>{item.title}</Text>
        </TouchableOpacity>
      )}
    />
  )
}
```

Writes are local-first and synced to Atlas automatically when the device is online.

## Summary

Atlas App Services with React Native provides authentication, data access rules, and offline-first sync through the `@realm/react` package. Wrap your app in `AppProvider` and `RealmProvider`, use hooks like `useQuery` and `useRealm` for reactive data access, and write to the local Realm database knowing that Device Sync will propagate changes to Atlas automatically.
