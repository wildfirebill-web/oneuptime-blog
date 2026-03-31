# How to Use MongoDB with Redwood.js

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Redwood.js, Full-Stack

Description: Integrate MongoDB with Redwood.js using Prisma's MongoDB connector, with schema definition, services, and GraphQL SDL for a full-stack app.

---

## Redwood.js and MongoDB

Redwood.js is a full-stack React framework with a GraphQL API layer powered by Prisma. Prisma's MongoDB connector allows Redwood to use MongoDB as the database, giving you type-safe queries, schema management, and the full Redwood development experience including generators, services, and cells.

## Project Setup

```bash
yarn create redwood-app my-app
cd my-app
```

Configure `.env`:

```text
DATABASE_URL="mongodb+srv://username:password@cluster0.mongodb.net/myapp?retryWrites=true&w=majority"
```

## Prisma Schema for MongoDB

Edit `api/db/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "mongodb"
  url      = env("DATABASE_URL")
}

model Post {
  id          String    @id @default(auto()) @map("_id") @db.ObjectId
  title       String
  slug        String    @unique
  body        String
  status      String    @default("draft")
  publishedAt DateTime?
  authorId    String    @db.ObjectId
  author      User      @relation(fields: [authorId], references: [id])
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
}

model User {
  id        String   @id @default(auto()) @map("_id") @db.ObjectId
  email     String   @unique
  name      String?
  posts     Post[]
  createdAt DateTime @default(now())
}
```

Push the schema:

```bash
yarn rw prisma db push
```

## Generating a Scaffold

```bash
yarn rw generate scaffold post
```

This generates SDL, services, and React components for full CRUD operations.

## Service Layer

`api/src/services/posts/posts.ts`:

```typescript
import { db } from 'src/lib/db'
import type { QueryResolvers, MutationResolvers } from 'types/graphql'

export const posts: QueryResolvers['posts'] = () => {
  return db.post.findMany({
    where: { status: 'published' },
    orderBy: { publishedAt: 'desc' },
    include: { author: true },
  })
}

export const post: QueryResolvers['post'] = ({ id }) => {
  return db.post.findUnique({ where: { id } })
}

export const createPost: MutationResolvers['createPost'] = ({ input }) => {
  return db.post.create({
    data: input,
  })
}

export const updatePost: MutationResolvers['updatePost'] = ({ id, input }) => {
  return db.post.update({
    where: { id },
    data: input,
  })
}

export const deletePost: MutationResolvers['deletePost'] = ({ id }) => {
  return db.post.delete({ where: { id } })
}
```

## GraphQL SDL

`api/src/graphql/posts.sdl.ts`:

```typescript
export const schema = gql`
  type Post {
    id: String!
    title: String!
    slug: String!
    body: String!
    status: String!
    publishedAt: DateTime
    author: User!
    createdAt: DateTime!
  }

  type Query {
    posts: [Post!]! @requireAuth
    post(id: String!): Post @requireAuth
  }

  type Mutation {
    createPost(input: CreatePostInput!): Post! @requireAuth
    updatePost(id: String!, input: UpdatePostInput!): Post! @requireAuth
    deletePost(id: String!): Post! @requireAuth
  }

  input CreatePostInput {
    title: String!
    slug: String!
    body: String!
    authorId: String!
  }

  input UpdatePostInput {
    title: String
    slug: String
    body: String
    status: String
    publishedAt: DateTime
  }
`
```

## Summary

Redwood.js integrates with MongoDB through Prisma's MongoDB connector. Define your data model in `schema.prisma` using the MongoDB-specific `@db.ObjectId` type for IDs, push the schema with `prisma db push`, and use Redwood's generators to scaffold services and SDL. The Prisma client provides type-safe MongoDB queries that fit naturally into Redwood's service layer pattern.
