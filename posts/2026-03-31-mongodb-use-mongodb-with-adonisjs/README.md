# How to Use MongoDB with AdonisJS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, AdonisJS, Node.js

Description: Integrate MongoDB with AdonisJS using the Lucid MongoDB provider or Mongoose, with models, migrations, and query examples for a REST API.

---

## Setting Up AdonisJS with MongoDB

AdonisJS is a full-featured MVC framework for Node.js. While it uses Lucid ORM with SQL databases by default, you can integrate MongoDB using either the community Lucid MongoDB adapter or Mongoose directly.

## Option 1: Using Mongoose with AdonisJS

Install dependencies:

```bash
npm install mongoose @adonisjs/core
```

Create a MongoDB service provider at `providers/MongoProvider.ts`:

```typescript
import type { ApplicationContract } from '@ioc:Adonis/Core/Application'
import mongoose from 'mongoose'

export default class MongoProvider {
  constructor(protected app: ApplicationContract) {}

  public async boot() {
    const uri = this.app.config.get('database.mongo.uri')
    await mongoose.connect(uri, {
      maxPoolSize: 10,
      serverSelectionTimeoutMS: 5000,
    })
    console.log('MongoDB connected')
  }

  public async shutdown() {
    await mongoose.disconnect()
  }
}
```

Register it in `adonisrc.ts`:

```typescript
providers: [
  './providers/MongoProvider',
]
```

## Defining a Mongoose Model

Create `app/Models/User.ts`:

```typescript
import mongoose, { Schema, Document } from 'mongoose'

export interface IUser extends Document {
  name: string
  email: string
  createdAt: Date
}

const UserSchema = new Schema<IUser>({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  createdAt: { type: Date, default: Date.now },
})

export const User = mongoose.model<IUser>('User', UserSchema)
```

## Controller with MongoDB

Create `app/Controllers/Http/UsersController.ts`:

```typescript
import { HttpContextContract } from '@ioc:Adonis/Core/HttpContext'
import { User } from 'App/Models/User'

export default class UsersController {
  public async index({ response }: HttpContextContract) {
    const users = await User.find({}).select('name email createdAt').lean()
    return response.ok(users)
  }

  public async store({ request, response }: HttpContextContract) {
    const { name, email } = request.only(['name', 'email'])

    const existing = await User.findOne({ email })
    if (existing) {
      return response.conflict({ message: 'Email already registered' })
    }

    const user = await User.create({ name, email })
    return response.created(user)
  }

  public async show({ params, response }: HttpContextContract) {
    const user = await User.findById(params.id).lean()
    if (!user) return response.notFound({ message: 'User not found' })
    return response.ok(user)
  }

  public async destroy({ params, response }: HttpContextContract) {
    await User.findByIdAndDelete(params.id)
    return response.noContent()
  }
}
```

## Routes

In `start/routes.ts`:

```typescript
import Route from '@ioc:Adonis/Core/Route'

Route.group(() => {
  Route.get('/', 'UsersController.index')
  Route.post('/', 'UsersController.store')
  Route.get('/:id', 'UsersController.show')
  Route.delete('/:id', 'UsersController.destroy')
}).prefix('/api/users')
```

## Configuration

Add MongoDB URI to `config/database.ts`:

```typescript
import Env from '@ioc:Adonis/Core/Env'

export default {
  mongo: {
    uri: Env.get('MONGO_URI', 'mongodb://localhost:27017/myapp'),
  },
}
```

In `.env`:

```text
MONGO_URI=mongodb://localhost:27017/myapp
```

## Summary

AdonisJS integrates with MongoDB cleanly through Mongoose as a service provider. Register a custom provider to handle connection lifecycle, define Mongoose models for schema validation, and use standard controller patterns for CRUD operations. This approach keeps MongoDB concerns separate from AdonisJS routing and middleware, giving you the structure of AdonisJS with MongoDB's flexible document model.
