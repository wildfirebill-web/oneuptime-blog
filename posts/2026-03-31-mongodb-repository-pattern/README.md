# How to Implement the Repository Pattern with MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Repository, Architecture, Mongoose, TypeScript

Description: Learn how to implement the repository pattern with MongoDB and Mongoose to decouple data access logic from business logic in Node.js applications.

---

The repository pattern wraps data access behind an interface, making your business logic independent of the database implementation. This makes testing easier (you can mock the repository) and keeps queries out of your controllers and services.

## Generic Base Repository

Define a generic interface that all repositories implement:

```typescript
// repositories/IRepository.ts
export interface IRepository<T> {
  findById(id: string): Promise<T | null>;
  findOne(filter: Partial<T>): Promise<T | null>;
  find(filter: Partial<T>, options?: { limit?: number; skip?: number }): Promise<T[]>;
  create(data: Partial<T>): Promise<T>;
  updateById(id: string, update: Partial<T>): Promise<T | null>;
  deleteById(id: string): Promise<boolean>;
  count(filter: Partial<T>): Promise<number>;
}
```

## Base Mongoose Repository

```typescript
// repositories/BaseRepository.ts
import { Model, Document, FilterQuery, UpdateQuery } from 'mongoose';
import { IRepository } from './IRepository';

export class BaseRepository<T extends Document> implements IRepository<T> {
  constructor(protected model: Model<T>) {}

  async findById(id: string): Promise<T | null> {
    return this.model.findById(id).lean() as Promise<T | null>;
  }

  async findOne(filter: FilterQuery<T>): Promise<T | null> {
    return this.model.findOne(filter).lean() as Promise<T | null>;
  }

  async find(
    filter: FilterQuery<T>,
    { limit = 20, skip = 0 }: { limit?: number; skip?: number } = {}
  ): Promise<T[]> {
    return this.model.find(filter).skip(skip).limit(limit).lean() as Promise<T[]>;
  }

  async create(data: Partial<T>): Promise<T> {
    return this.model.create(data) as Promise<T>;
  }

  async updateById(id: string, update: UpdateQuery<T>): Promise<T | null> {
    return this.model.findByIdAndUpdate(id, update, { new: true }).lean() as Promise<T | null>;
  }

  async deleteById(id: string): Promise<boolean> {
    const result = await this.model.deleteOne({ _id: id });
    return result.deletedCount === 1;
  }

  async count(filter: FilterQuery<T>): Promise<number> {
    return this.model.countDocuments(filter);
  }
}
```

## Concrete User Repository

```typescript
// repositories/UserRepository.ts
import { BaseRepository } from './BaseRepository';
import User, { IUser } from '../models/User';

export class UserRepository extends BaseRepository<IUser> {
  constructor() {
    super(User);
  }

  async findByEmail(email: string): Promise<IUser | null> {
    return this.model.findOne({ email: email.toLowerCase() }).lean() as Promise<IUser | null>;
  }

  async findActiveUsers(limit = 50): Promise<IUser[]> {
    return this.model
      .find({ status: 'active', emailVerified: true })
      .sort({ createdAt: -1 })
      .limit(limit)
      .lean() as Promise<IUser[]>;
  }

  async incrementLoginCount(userId: string): Promise<void> {
    await this.model.findByIdAndUpdate(userId, {
      $inc: { loginCount: 1 },
      $set: { lastLoginAt: new Date() },
    });
  }
}
```

## Concrete Post Repository

```typescript
// repositories/PostRepository.ts
import { BaseRepository } from './BaseRepository';
import Post, { IPost } from '../models/Post';

export class PostRepository extends BaseRepository<IPost> {
  constructor() {
    super(Post);
  }

  async findPublishedByAuthor(authorId: string): Promise<IPost[]> {
    return this.model
      .find({ authorId, published: true })
      .sort({ createdAt: -1 })
      .lean() as Promise<IPost[]>;
  }

  async findByTags(tags: string[], limit = 20): Promise<IPost[]> {
    return this.model
      .find({ tags: { $in: tags }, published: true })
      .limit(limit)
      .lean() as Promise<IPost[]>;
  }
}
```

## Using Repositories in a Service

Business logic lives in services, never directly in controllers:

```typescript
// services/UserService.ts
import { UserRepository } from '../repositories/UserRepository';
import { PostRepository } from '../repositories/PostRepository';

export class UserService {
  constructor(
    private userRepo: UserRepository,
    private postRepo: PostRepository
  ) {}

  async getUserProfile(userId: string) {
    const user = await this.userRepo.findById(userId);
    if (!user) throw new Error('User not found');

    const posts = await this.postRepo.findPublishedByAuthor(userId);
    return { ...user, postCount: posts.length, recentPosts: posts.slice(0, 5) };
  }
}
```

## Wiring Up in a Controller

```typescript
const userRepo = new UserRepository();
const postRepo = new PostRepository();
const userService = new UserService(userRepo, postRepo);

app.get('/api/users/:id', async (req, res) => {
  const profile = await userService.getUserProfile(req.params.id);
  res.json(profile);
});
```

## Summary

The repository pattern separates data access from business logic by wrapping Mongoose queries behind a typed interface. A generic `BaseRepository` handles common CRUD operations, while concrete repositories add domain-specific query methods. Business logic in services only interacts with repository interfaces, making unit testing straightforward by substituting mock repositories.
