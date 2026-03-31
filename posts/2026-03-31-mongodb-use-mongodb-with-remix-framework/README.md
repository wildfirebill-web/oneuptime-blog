# How to Use MongoDB with Remix Framework

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Remix, Loader, Action, Mongoose

Description: Learn how to connect MongoDB to a Remix application using loaders and actions for server-side data fetching and mutations with a cached connection.

---

## Introduction

Remix runs all data loading and mutations on the server through `loader` and `action` functions. This makes it straightforward to connect MongoDB - your database credentials stay server-side and you can query directly from loaders without an intermediate API layer.

## Installation

```bash
npm install mongoose
```

## Connection Helper

```typescript
// app/db.server.ts
import mongoose from 'mongoose';

let isConnected = false;

export async function connectDB(): Promise<void> {
  if (isConnected) return;

  await mongoose.connect(process.env.MONGODB_URI!, {
    bufferCommands: false,
  });
  isConnected = true;
  console.log('MongoDB connected');
}
```

## Mongoose Model

```typescript
// app/models/post.server.ts
import mongoose, { Schema, model, models } from 'mongoose';

export interface IPost {
  _id:       string;
  title:     string;
  body:      string;
  published: boolean;
  createdAt: Date;
}

const PostSchema = new Schema<IPost>(
  { title: String, body: String, published: { type: Boolean, default: false } },
  { timestamps: true }
);

export const Post = models.Post ?? model<IPost>('Post', PostSchema);
```

## Loader - Reading Data

```typescript
// app/routes/posts._index.tsx
import { json, type LoaderFunctionArgs } from '@remix-run/node';
import { useLoaderData }                 from '@remix-run/react';
import { connectDB }                     from '~/db.server';
import { Post }                          from '~/models/post.server';

export async function loader({ request }: LoaderFunctionArgs) {
  await connectDB();
  const posts = await Post.find({ published: true })
    .sort({ createdAt: -1 })
    .limit(10)
    .lean();
  return json({ posts });
}

export default function PostsIndex() {
  const { posts } = useLoaderData<typeof loader>();
  return (
    <ul>
      {posts.map((p) => <li key={p._id}>{p.title}</li>)}
    </ul>
  );
}
```

## Action - Mutating Data

```typescript
// app/routes/posts.new.tsx
import { redirect, type ActionFunctionArgs } from '@remix-run/node';
import { Form }                               from '@remix-run/react';
import { connectDB }                          from '~/db.server';
import { Post }                               from '~/models/post.server';

export async function action({ request }: ActionFunctionArgs) {
  await connectDB();
  const formData = await request.formData();
  await Post.create({
    title:     formData.get('title') as string,
    body:      formData.get('body')  as string,
    published: formData.get('published') === 'on',
  });
  return redirect('/posts');
}

export default function NewPost() {
  return (
    <Form method="post">
      <input name="title" placeholder="Title" required />
      <textarea name="body" placeholder="Body" />
      <label><input type="checkbox" name="published" /> Publish</label>
      <button type="submit">Create Post</button>
    </Form>
  );
}
```

## Summary

Remix's `loader` and `action` functions run exclusively on the server, making them the natural place to call `connectDB()` and query MongoDB. Use a singleton connection flag to avoid reconnecting on every request. The `Post.find().lean()` pattern returns plain objects that JSON-serialize cleanly through Remix's `json()` helper for use in components.
