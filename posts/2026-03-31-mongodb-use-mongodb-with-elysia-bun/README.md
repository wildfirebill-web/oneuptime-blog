# How to Use MongoDB with Elysia (Bun)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Bun, Elysia

Description: Build a fast REST API with Elysia on Bun runtime connected to MongoDB using Mongoose, with type-safe route handlers and schema validation.

---

## Elysia and MongoDB

Elysia is a high-performance web framework designed for Bun, the fast JavaScript runtime. It features end-to-end type safety through its schema system, making it a natural fit with Mongoose for MongoDB integration. Bun's fast startup time and native TypeScript support make the combination well-suited for microservices.

## Project Setup

```bash
bun create elysia my-api
cd my-api
bun add mongoose
```

## MongoDB Connection

Create `src/db.ts`:

```typescript
import mongoose from 'mongoose'

const MONGO_URI = process.env.MONGO_URI ?? 'mongodb://localhost:27017/elysiaapp'

export async function connectDB() {
  await mongoose.connect(MONGO_URI, {
    maxPoolSize: 10,
    serverSelectionTimeoutMS: 5000,
  })
  console.log('MongoDB connected')
}

export async function disconnectDB() {
  await mongoose.disconnect()
}
```

## Defining a Mongoose Model

Create `src/models/Product.ts`:

```typescript
import mongoose, { Schema, Document } from 'mongoose'

export interface IProduct extends Document {
  name: string
  price: number
  category: string
  stock: number
  createdAt: Date
}

const ProductSchema = new Schema<IProduct>({
  name:     { type: String, required: true, index: true },
  price:    { type: Number, required: true, min: 0 },
  category: { type: String, required: true, index: true },
  stock:    { type: Number, required: true, default: 0 },
  createdAt: { type: Date, default: Date.now },
})

export const Product = mongoose.model<IProduct>('Product', ProductSchema)
```

## Building the Elysia API

In `src/index.ts`:

```typescript
import { Elysia, t } from 'elysia'
import { connectDB } from './db'
import { Product } from './models/Product'

await connectDB()

const app = new Elysia()
  .get('/products', async () => {
    return Product.find({}).lean()
  })

  .get('/products/:id', async ({ params, set }) => {
    const product = await Product.findById(params.id).lean()
    if (!product) {
      set.status = 404
      return { error: 'Product not found' }
    }
    return product
  })

  .post('/products', async ({ body, set }) => {
    const product = await Product.create(body)
    set.status = 201
    return product
  }, {
    body: t.Object({
      name:     t.String({ minLength: 1 }),
      price:    t.Number({ minimum: 0 }),
      category: t.String({ minLength: 1 }),
      stock:    t.Optional(t.Number({ minimum: 0 })),
    }),
  })

  .patch('/products/:id', async ({ params, body, set }) => {
    const updated = await Product.findByIdAndUpdate(
      params.id,
      { $set: body },
      { new: true, runValidators: true }
    )
    if (!updated) {
      set.status = 404
      return { error: 'Product not found' }
    }
    return updated
  }, {
    body: t.Object({
      price: t.Optional(t.Number({ minimum: 0 })),
      stock: t.Optional(t.Number({ minimum: 0 })),
    }),
  })

  .delete('/products/:id', async ({ params, set }) => {
    await Product.findByIdAndDelete(params.id)
    set.status = 204
  })

  .listen(3000)

console.log(`Elysia running at http://localhost:${app.server?.port}`)
```

## Running the Application

```bash
bun run src/index.ts
```

## Testing with curl

```bash
# Create a product
curl -X POST http://localhost:3000/products \
  -H 'Content-Type: application/json' \
  -d '{"name":"Widget","price":9.99,"category":"tools","stock":100}'

# List all products
curl http://localhost:3000/products

# Update stock
curl -X PATCH http://localhost:3000/products/PRODUCT_ID \
  -H 'Content-Type: application/json' \
  -d '{"stock":50}'
```

## Summary

Elysia on Bun pairs well with MongoDB through Mongoose, offering fast startup times, native TypeScript, and built-in request body validation through Elysia's schema system. Connect MongoDB in an async bootstrap, define Mongoose models for schema validation and indexing, and use Elysia's typed route handlers to build a type-safe REST API. Bun's speed makes this stack particularly effective for high-throughput microservices.
